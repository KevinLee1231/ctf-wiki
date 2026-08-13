# Vexed

## 题目简述

服务接收最长 `0x300` 字节的 Base64 shellcode，用 Capstone 按 x86-64 反汇编，要求每条初始指令都属于 AVX2 指令组，并拒绝助记符中包含 `mov` 的指令。校验通过后，代码被复制到一段同时可读、可写、可执行的 `mmap` 内存并直接调用。目标是在静态检查只看到 AVX2 的前提下执行普通系统调用并读取 `flag.txt`。

## 解题过程

过滤器只检查执行前的字节流，没有限制 AVX2 指令的内存写操作，也没有在执行期间重新反汇编代码。RWX 映射因此允许第一阶段用 AVX2 把后续区域改写成任意第二阶段 shellcode。

官方解法先准备普通的 `open`、`read`、`write`、`exit` shellcode，再把它按 4 字节切块。初始代码中用 `vpaddd` 指令编码的位移字段保存这些块的低 31 位；若块的最高位为 1，则先用 `vpslld` 构造 `0x80000000`。随后把数据装入 YMM 寄存器，并用允许的 AVX2 存储指令覆盖 `stage2`：

```asm
vpxor ymm0, ymm0, ymm0
vpaddd ymm1, ymm0, YMMWORD PTR [rip + encoded_one]
vpslld ymm1, ymm1, 31

; 对每个 4 字节块重复：
vpaddd ymm2, ymm0, YMMWORD PTR [rip + encoded_chunk]
; 最高位为 1 时改为以 ymm1 为基值
vextracti128 XMMWORD PTR [rip + stage2 + offset], ymm2, 0
```

`stage2` 的初始占位内容本身也由可通过检查的 `vpaddd` 指令组成。第一阶段重写完成后自然向下执行，此时该区域已经变成常规 x86-64 shellcode。第二阶段的逻辑为：

```asm
; open("./flag.txt", O_RDONLY)
; read(fd, rsp, 0x100)
; write(1, rsp, bytes_read)
; exit(0)
```

将生成后的整段字节流 Base64 编码提交，服务只会看到合法的非 `mov` AVX2 初始指令；运行时自修改后则执行未被检查的系统调用，输出：

```text
grey{vexed,VEXed_i_tell_you!}
```

## 方法总结

- 核心技巧：利用 RWX 代码页与只检查初始字节的静态过滤，构造 AVX2 自修改第一阶段。
- 识别信号：指令白名单若允许向代码页写内存，却不验证控制流或执行后的字节，通常可以转为 staged shellcode。
- 复用要点：第二阶段占位字节同样必须能被初始反汇编完整接受；重叠写入时还要保证最终落地字节顺序正确。
