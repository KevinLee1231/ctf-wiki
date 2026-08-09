# low_re

## 题目简述

程序要求输入恰好 17 个字符，依次执行固定密钥 RC4、以每个结果字节为种子的 glibc `rand()`，再把十进制随机数连续做 2560 轮 SHA-256。校验按字符从左到右进行，遇到首个错误立即退出：

```python
for i in range(17):
    for _ in range(2560):
        flag[i] = hashlib.sha256(flag[i].encode()).hexdigest()
    if flag[i] != ciphertext[i]:
        sys.exit()
```

这种“正确前缀越长，执行工作量越大”的控制流制造了指令数侧信道。官方 PDF 共 2 页，逐页视觉核对后确认只有说明文字和一段跨页代码，没有不可替代的图形信息；本篇已完整转写其插桩思路，并修正代码截图中省略的稳定性条件。

## 解题过程

### 选择观测量

普通墙钟时间会受调度、进程启动和库函数影响。使用 Intel Pin 或 DynamoRIO 统计动态指令数更稳定，但必须只统计主程序模块；如果把 Python 运行库、加载器等模块也计入，后几个字符的噪声会淹没一次校验差异。题目特意设置 2560 轮 SHA-256，正是为了放大“继续检查一个字符”对应的指令差。

构造候选时保持总长度为 17。已恢复前缀固定，只枚举下一个可打印字符，其余位置用任意字符填充。候选在当前位置正确时会继续执行下一轮哈希，因此取指令数最大的字符：

```python
from concurrent.futures import ThreadPoolExecutor
from string import printable

def recover(test_count, length=17):
    alphabet = "".join(ch for ch in printable if 32 <= ord(ch) < 127)
    known = ""
    with ThreadPoolExecutor(max_workers=8) as pool:
        while len(known) < length:
            samples = []
            for candidate in alphabet:
                probe = known + candidate + "i" * (length - len(known) - 1)
                samples.append(probe)

            counts = list(pool.map(test_count, samples))
            known += alphabet[max(range(len(counts)), key=counts.__getitem__)]
            print(known)
    return known
```

`test_count()` 的职责是通过插桩器启动一次 `low_re.exe`、发送候选、解析本次主模块指令计数并关闭进程。Pin 的调用形式可以写为：

```text
pin.exe -t instruction-counter.dll -- low_re.exe
```

DynamoRIO 客户端则应启用类似 `-only_from_app` 的过滤。每轮最好对最高的两三个候选重复测量并取中位数，避免单次异常把错误字符带入后续前缀。

### 有源码时的直接校验表路线

当前归档仓库还提供了 Python 源码，因此可以避开侧信道：RC4 后的每个字节只可能是 0 至 255，`srand(byte)`、第一次 `rand()` 和 2560 轮 SHA-256 都是确定函数。预计算 256 个输入到最终摘要的反向表，就能逐项恢复 RC4 密文字节，再用密钥 `Sycl0ver` 的 RC4 密钥流异或还原 17 字节输入。

```python
from hashlib import sha256

digest_to_byte = {}
for value in range(256):
    linux_srand(value)
    state = str(linux_rand())
    for _ in range(2560):
        state = sha256(state.encode()).hexdigest()
    digest_to_byte[state] = value

rc4_cipher = bytes(digest_to_byte[item] for item in ciphertext)
flag = rc4_crypt("Sycl0ver", rc4_cipher)
```

这里的 `linux_srand`、`linux_rand` 和 RC4 必须保留题目源码的实现细节，不能用 Python `random` 或其它 libc 版本代替。尤其是题目 RC4 的 KSA 循环结束后变量 `i` 保持为 255，PRGA 第一轮因此从 `i=0` 开始；换成标准库 RC4 会得到错误结果。

对源码中的 17 个摘要执行反查，得到 RC4 输出字节：

```text
f1 08 68 48 53 80 68 36 e2 04 c8 a8 e5 96 17 3c 03
```

再用题目原样实现的 RC4 逆变换，恢复程序要求输入的 17 字节：

```text
S1deCh4nnelAtt@ck
```

把它重新送入 `low_re.py`，17 次摘要比较全部通过并输出 `you are right`。仓库程序验证的是花括号内部字符串，并不自行打印比赛平台 flag；按 SCTF2021 的通用提交格式，提交值为：

```text
SCTF{S1deCh4nnelAtt@ck}
```

## 方法总结

本题的核心不是破解 SHA-256，而是利用逐字符早退产生的工作量差异。没有源码时，通过只统计主模块指令数逐位恢复；有源码时，输入域仅 256 个字节，反而可以为整条确定变换建立反向表。两条路线都依赖同一事实：每个位置独立转换且比较失败立即退出。复现动态侧信道时，应固定输入长度、过滤库指令、对边界候选重复采样，并在每一步验证正确前缀确实令计数增加；走源码反查路线时则必须保留自定义 glibc `rand` 与非标准 RC4 的全部细节。
