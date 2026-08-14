# CakeCTF2021 ALDRYA

## 题目简述

服务接收一个 ELF 和对应的 ALDRYA 签名。签名文件先保存 ELF 的 0x100 字节分块数，随后为每块保存一个 32 位自定义哈希。验证器检查 ELF 魔数、总长度和各块哈希，全部通过后直接 `execv` 上传的 ELF。

问题在于哈希只有旋转与异或：

```c
hash = 0x20210828;
for (int i = 0; i < 0x100; i++)
    hash = ROR32(hash ^ chunk[i]);
```

这个变换没有密码学抗碰撞性。可以保留恶意 ELF 每块的有效前缀，再求出 32 个可控尾字节，使该块与官方 `sample.elf` 的签名相同。

## 解题过程

### 解析签名与验证边界

`sample.aldrya` 的格式是一个小端 32 位块数 $n$，后跟 $n$ 个小端 32 位哈希。验证器允许 ELF 大小满足

$$
(n-1)\times0x100<\text{size}\le n\times0x100,
$$

并对最后不足一块的内容以零补齐后计算哈希。最方便的做法是生成恰好 $n$ 个完整块。

### 构造能读取 flag 的最小 ELF

先汇编一个尽可能短的 x86-64 ELF，例如执行 `open/read/write` 读取 `flag.txt`。体积越小，需要真正碰撞的前几块就越少；恶意 ELF 用完后的剩余分块可以直接复制 `sample.elf` 对应内容。

对每个含有恶意内容的分块，保留前 224 字节，把最后 32 字节设为 8 位 Z3 变量。按验证器原样建立 32 位位向量约束：

```python
from z3 import BitVec, RotateRight, Solver, ZeroExt, sat

def ror32(value):
    if isinstance(value, int):
        return ((value >> 1) | ((value & 1) << 31)) & 0xffffffff
    return RotateRight(value, 1)

def collide(chunk, wanted):
    prefix = bytearray(chunk[:-32])
    suffix = [BitVec(f"p{i}", 8) for i in range(32)]
    h = 0x20210828
    for b in prefix:
        h = ror32(h ^ b)
    for b in suffix:
        h = ror32(h ^ ZeroExt(24, b))

    solver = Solver()
    solver.add(h == wanted)
    assert solver.check() == sat
    model = solver.model()
    return bytes(prefix) + bytes(model[b].as_long() for b in suffix)
```

32 次单比特循环移位使每个可控字节的影响扩散到不同位置，256 个自由比特足以满足一个 32 位目标等式。

### 组装并提交

逐块使用签名中的目标哈希生成碰撞块；恶意 ELF 结束后，复制样本对应块以保持其哈希。最终文件仍以合法 ELF 头开头，大小也在签名声明范围内，因此验证通过并被执行，读出：

```text
CakeCTF{jUst_cH3ck_SHA256sum_4nd_7h47's_f1n3}
```

## 方法总结

- 自定义哈希仅由可逆线性操作组成时，短输出碰撞通常可以直接建模为 SAT/SMT 约束。
- 分块签名只验证每块摘要，不验证文件整体语义；可控块与原样复制块可以组合成新的可执行文件。
- 上传后直接执行会把“弱完整性校验”放大成远程代码执行，签名算法必须使用成熟的密码学签名而非自制滚动哈希。
