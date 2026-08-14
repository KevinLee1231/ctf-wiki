# CakeCTF2021 improvisation

## 题目简述

题目用一个 64 位 Fibonacci LFSR 生成比特流，再与 flag 逐位异或。反馈位取当前状态的第 $0,1,3,4$ 位异或，状态每轮右移一位，并把反馈位放到最高位。

LFSR 本身是线性的，而且 flag 固定以 `CakeCTF{` 开头。这个已知前缀正好有 8 字节，即 64 位，足以恢复完整初始状态。

## 解题过程

### 对齐题目的位序

加密端把明文按小端整数读入，逐轮取最低位参与异或，同时用左移累积输出。因此直接把输出十六进制当普通字节串处理会遇到整体位序相反的问题。官方脚本先反转密文整数的二进制表示，再左移一位，得到与 LFSR 输出顺序一致的 `cc`：

```python
with open("distfiles/output.txt", "r", encoding="utf-8") as f:
    c = int(f.read(), 16)

c = int(bin(c)[2:][::-1], 2)
cc = c << 1
```

### 用已知前缀恢复种子

设前 64 个密文位组成 $C_0$，已知前缀的小端整数为 $P$，首轮 64 个密钥流位就是初始 LFSR 状态的 64 位，因此

$$
\text{seed}=C_0\oplus P.
$$

对应代码为：

```python
prefix = int.from_bytes(b"CakeCTF{", "little")
seed = (cc & ((1 << 64) - 1)) ^ prefix
```

恢复种子后，按题目相同的反馈规则重新生成密钥流并逐位解密：

```python
def lfsr(seed):
    state = seed
    while True:
        yield state & 1
        bit = ((state >> 0) ^ (state >> 1) ^
               (state >> 3) ^ (state >> 4)) & 1
        state = (state >> 1) | (bit << 63)

stream = lfsr(seed)
m = 0
work = cc
while work:
    m = (m << 1) | ((work & 1) ^ next(stream))
    work >>= 1

plain = int(bin(m)[2:][::-1], 2).to_bytes(64, "little").rstrip(b"\x00")
print(plain)
```

仓库附件实测得到：

```text
CakeCTF{d0n't_3xp3c7_s3cur17y_2_LSFR}
```

## 方法总结

- 64 位已知明文泄露了 64 位 LFSR 的全部初始状态，后续密钥流随之完全可预测。
- 做位流题时必须逐项确认整数端序、取位方向和输出累积方向；这道题最容易错在整体位序。
- LFSR 适合纠错码和伪随机序列，不应直接作为已知明文环境中的流密码。
