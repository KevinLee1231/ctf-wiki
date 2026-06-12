# VirtualNPU

## 题目简述

题目把 flag 校验放进 CUDA fatbin 中的虚拟 NPU。宿主程序负责读入 flag、解密内置 NPU 字节码并调度 GPU kernel；真正比较逻辑在 NPU 字节码里。解法不需要完整模拟 NPU，只要恢复字节码、提取比较常量并逆转三层可逆变换。

## 解题过程

宿主程序是 PE32+ 可执行文件，包含 `.nv_fatb` 段。用 `strings` 或 PE 工具可以看到：

```text
VirtualNPU Flag Verifier
.nv_fatb
_Z16npu_cycle_kernelPhPjjPyS0_
```

优先用 `cuobjdump` 查看 fatbin：

```bash
cuobjdump --dump-elf virtualnpu.exe
cuobjdump --dump-ptx virtualnpu.exe
```

可以确认：

```text
只有一个可见 kernel: _Z16npu_cycle_kernelPhPjjPyS0_
常量符号 AES_SBOX 大小为 0x100
共享内存很大，register count = 255，frame size = 0xdd0
PTX 中直接保留 .const .b8 AES_SBOX[256]
```

这说明 CUDA kernel 更像一个自定义 VM/NPU 解释器，核心算法仍围绕字节级状态和 AES S-box。

宿主层会从 PE 中取出加密字节码，用固定 salt 派生 32 字节 RC4 key：

```python
def derive_rc4_key():
    salt = b"VirtualNPU_2026_SALT"
    return bytes(salt[i % len(salt)] ^ ((i * 0x37) & 0xff) for i in range(32))
```

随后执行 RC4，并丢弃前 `0x200` 字节 keystream，再解密字节码。忘记 drop-512 会导致整段程序错误。定位密文时可以利用第一条明文 bundle 的标量槽特征：`MOV_IMM R14, 13` 编码为小端 `0x05c0000d`，先用派生出的 keystream 算出其密文前 4 字节，再在 EXE 中搜索。

解密后得到 768 条虚拟指令，每条 16 字节：

```text
dword0  标量指令
dword1  向量指令
dword2  张量指令
dword3  扩展/保留
```

比较阶段有 38 次重复结构：

```text
LBU R11, <addr>(R0)
MOV_IMM R12, <imm>
NOP
BNE R11, R12, wrong
```

`MOV_IMM R12, <imm>` 的立即数字段就是最终比较目标 `ct3`。

扫描标量槽时使用的掩码为：

```python
MOV_R12_MASK  = 0xffff0000
MOV_R12_MATCH = (0x01 << 26) | (12 << 21)  # 0x05800000
```

字节码中的变换分三层。第一层是基于 AES S-box 和滚动状态的异或流：

```text
S    = 0x31415926
sb   = AES_SBOX[S & 0xff]
S    = ROL64(S, 13) ^ sb
ks_i = S & 0xff
ct1[i] = flag[i] ^ ks_i
```

第二层是矩阵阶段 `MSBOX`，对字节做可逆变换：

```text
偶数字节: ct2[j] = ~ct1[j] & 0xff
奇数字节: ct2[j] = ((ct1[j] ^ 0xa5) + 0x37) & 0xff
```

逆变换为：

```text
偶数字节: ct1[j] = ~ct2[j] & 0xff
奇数字节: ct1[j] = ((ct2[j] - 0x37) ^ 0xa5) & 0xff
```

第三层会把 `AES_SBOX` 前 64 字节放到向量区，再对第二层结果做异或：

```text
vkey = AES_SBOX[:64]
ct3[j] = ct2[j] ^ vkey[j]
```

完整逆推链路为：

```text
compare_target = ct3
ct2 = compare_target ^ AES_SBOX[:38]
ct1 = inverse_MSBOX(ct2)
flag = ct1 ^ keystream
```

解出：

```text
ACTF{470a43092156babd744087029e4f9297}
```

运行 `virtualnpu.exe` 后输出 `ACTF{...}`，验证了虚拟 NPU 逻辑还原结果。

## 方法总结

- 核心技巧：从 PE 的 CUDA fatbin 中恢复自定义 NPU 字节码，提取比较常量后逆转三层可逆变换。
- 识别信号：宿主程序只读入和调度，而 `.nv_fatb`、GPU kernel、bytecode 和比较常量同在时，应转向 GPU/VM 逆向。
- 复用要点：先确认字节码解密正确，再提比较目标；不要一开始完整模拟 NPU，能静态抽象的变换应直接求逆。
