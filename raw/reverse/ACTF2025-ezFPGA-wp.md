# ezFPGA

## 题目简述

附件包含 SystemVerilog 加密模块和一份仿真波形 `Testbench.vcd`。flag 先补零到 39 字节，经过长度为 4 的一维卷积得到 36 字节，再重排为 $6\times6$ 矩阵并右乘固定矩阵，最后进入使用密钥 `eclipsky` 的修改版 RC4。目标是从 VCD 恢复 36 字节密文并逐层逆运算。

这道题不要求掌握 FPGA 物理结构，关键是正确理解时序波形和 `logic [7:0]` 的截断语义：所有卷积、矩阵和流加密加法都在模 $256$ 下进行。

## 解题过程

### 从 VCD 恢复密文

VCD 是按事件记录的：一个信号只有在值发生变化时才会出现新记录。不能简单收集 `cypher` 的所有变更值，否则相邻两个相同字节只会被记录一次。应根据时钟边沿和状态机进入 `S3` 的时间，每个输出周期采样一次，并在没有新事件时沿用上一次值；也可以直接用 GTKWave 检查相应 36 个周期。

恢复出的密文为：

```python
cypher = [
    173, 0, 192, 159, 22, 23, 236, 37, 37, 31, 18, 226,
    127, 159, 55, 83, 18, 186, 141, 56, 96, 20, 27, 49,
    142, 19, 226, 86, 10, 26, 37, 185, 128, 115, 138, 96,
]
```

附件之间还有一个需要说明的不一致：当前 `encryptor.sv` 的复位分支写成 `state <= S1`，会跳过 `S0` 中对 `ba[i]=i` 的初始化；但随题 VCD 的首个非零输出约在第 550 个周期，时间恰好包含 `S0` 的 256 轮初始化、`S1` 的 256 轮 KSA 和 `S2` 的 36 轮加密，官方解密结果也对应标准 RC4 状态表。因此 VCD 来自预期的 `state <= S0` 版本。若要用现有源码重新仿真，应把复位状态改回 `S0`；解题时则以随题 VCD 为最终证据。

### 逆修改版 RC4

状态机的 KSA 和 PRGA 使用标准 RC4 的状态更新，密钥为 ASCII 字符串 `eclipsky`。不同点只在输出组合：标准 RC4 使用异或，而题目执行模 $256$ 加法：

$$
c_i=(m_i+k_i)\bmod256.
$$

因此解密应减去密钥流：

```python
def undo_rc4_add(key, data):
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) & 0xff
        S[i], S[j] = S[j], S[i]

    i = j = 0
    out = []
    for c in data:
        i = (i + 1) & 0xff
        j = (j + S[i]) & 0xff
        S[i], S[j] = S[j], S[i]
        k = S[(S[i] + S[j]) & 0xff]
        out.append((c - k) & 0xff)
    return out

mixed = undo_rc4_add(b"eclipsky", cypher)
```

### 逆 $6\times6$ 矩阵

源码将卷积结果 `ac` 每 6 字节分为一行，并计算：

$$
AE=AC\cdot AD\pmod{256}.
$$

仓库解题脚本已经给出 $AD^{-1}$。使用整数矩阵乘法后立即模 $256$：

```python
AD_inv = np.array([
    [167, 214,  99, 125,  29, 213],
    [180,  50,  35, 197, 253, 150],
    [203,  84,  50, 206, 152, 150],
    [199, 243,  73, 104, 234,  69],
    [163,  84, 170, 216, 107, 189],
    [127, 219, 196, 158, 221, 222],
], dtype=np.int64)

conv = (np.array(mixed).reshape(6, 6) @ AD_inv) % 256
conv = conv.reshape(-1)
```

这里不能使用普通浮点逆矩阵后四舍五入，因为电路中的等式属于 $\mathbb Z/256\mathbb Z$。应额外验证 `AD @ AD_inv % 256` 为单位矩阵。

### 逐字节逆卷积

flag 被补零到 39 字节，卷积核为：

```text
[11, 4, 5, 14]
```

第 $i$ 个输出满足：

$$
ac_i=11a_i+4a_{i+1}+5a_{i+2}+14a_{i+3}\pmod{256},
\qquad 0\le i<36.
$$

已知 flag 前缀为 `ACTF{`。从 $i=2$ 开始时，$a_2$、$a_3$、$a_4$ 已知，方程中只有 $a_5$ 未知；找到它后移动到 $i=3$，就能逐字节向后恢复。用 flag 常见字符集枚举即可：

```python
flag = "ACTF{"
alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_@{}"
kernel = [11, 4, 5, 14]

for i in range(2, 36):
    for ch in alphabet:
        trial = flag + ch
        value = sum(ord(trial[i + j]) * kernel[j] for j in range(4)) & 0xff
        if value == conv[i]:
            flag = trial
            break
    if flag.endswith("}"):
        break
```

最终恢复：

```text
ACTF{RC4_4nd_FPGA_w4lk_1nt0_4_b4r}
```

## 方法总结

解题顺序是“VCD 定周期采样—RC4 密钥流相减—模 $256$ 矩阵求逆—利用已知前缀滚动逆卷积”。VCD 的事件压缩和 8 位硬件类型的自然截断是两个最容易忽略的点。分析 HDL 时，应先把连续赋值和状态机整理成数学变换，再明确每一步所在的数值域；这样无需搭建真实 FPGA 环境，也能完整复现算法。
