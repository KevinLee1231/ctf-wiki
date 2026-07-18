# week2Stupid repeater

## 题目简述

公开官方 WP 将本题与 `N1k0la's love` 一并标记为格式化字符串题，但没有公开 EXP；当前仓库同样缺少二进制。因此能够确认的核心只有“不可信输入被当作格式串”，无法可靠断言具体偏移、GOT 项或后门地址。

## 解题过程

拿到原附件后，先用普通字符串与 `%p` 对比输出，确认漏洞是否位于 `printf(user_input)` 一类调用。接着批量输出参数，寻找输入标记在栈上的位置：

```python
from pwn import *

io = process("./main")
probe = b"MARK" + b".".join(f"%{index}$p".encode() for index in range(1, 30))
io.sendline(probe)
print(io.recvall())
```

若输出中出现 `MARK` 对应的十六进制值，即可确定后续位置参数的基准。然后检查：

- `%s` 是否可以从指定地址读取 GOT 或字符串；
- `%n`、`%hn`、`%hhn` 是否可用；
- RELRO 是否允许改写 GOT；
- 程序是否重复读取，从而可以先泄露、再写入；
- PIE/ASLR 是否要求先恢复程序或 libc 基址。

确定真实写入目标后，使用 `fmtstr_payload(offset, {target: value}, write_size="short")` 生成 payload，并在调试器中验证每段写入值。若程序会重复输出固定前缀，必须把已输出字节数计入 `%n`，否则写入会整体偏移。

## 方法总结

本题现存资料只能支持格式化字符串的通用利用链，不能支持一个带具体常量的最终 EXP。与其复制无法验证的模板，更可靠的补全方式是取得原二进制后记录四项实测证据：参数 offset、保护状态、写入目标、写入后的控制流；这四项也是后续恢复本题完整 WP 所必需的材料。
