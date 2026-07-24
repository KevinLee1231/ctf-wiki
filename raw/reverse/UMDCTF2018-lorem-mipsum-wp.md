# UMDCTF 2018 - Lorem Mipsum

## 题目简述

题目提供 MIPS 汇编 `lorem_mipsum.s` 和 1024 字节密文 `data.enc`。汇编中既有 32 位 LFSR 异或函数 `ec`，也有按 8 字节分组转置位矩阵的函数 `s`。不过，必须以附件的可复现结果为准：`data.enc` 只需逆转 LFSR 异或即可还原明文，再套用 `s` 会破坏结果。

## 解题过程

汇编中的 LFSR 初始状态为：

```text
938214802
```

`hurrrrr` 每产生一位，先把当前状态最低位移入输出字最高位，再用状态第 `0、1、5、8、12、31` 位的异或作为新状态最高位。连续执行 32 次便得到一个密钥字：

```python
def lfsr_word(state):
    output = 0
    for _ in range(32):
        output = (output >> 1) | ((state & 1) << 31)
        feedback = 0
        for bit in (0, 1, 5, 8, 12, 31):
            feedback ^= (state >> bit) & 1
        state = ((state >> 1) | (feedback << 31)) & 0xffffffff
    return state, output
```

按大端序读取 `data.enc` 中的 32 位字，并逐字与密钥流异或：

```python
from pathlib import Path

data = bytearray(Path("data.enc").read_bytes())
state = 938214802

for offset in range(0, len(data), 4):
    state, key = lfsr_word(state)
    word = int.from_bytes(data[offset:offset + 4], "big")
    data[offset:offset + 4] = (word ^ key).to_bytes(4, "big")

plaintext = bytes(data)
print(plaintext.split(b"\x00", 1)[0].decode())
```

输出开头是连续明文：

```text
UMDCTF-{l1n3_sH1ft_F33dB@cK_r3Gi5t3rz_&_p3rMu7at10nz}
```

该字符串的 SHA-256 为：

```text
4123e561b24323371ca4a4481ad9daf4f5e41f59f08fcd21d535516a6be62b0c
```

它与 `README.md` 保存的官方摘要完全一致。

需要特别说明源码与工件的差异：`main` 在 `ec` 后还会调用 `s`，但对附件执行相应的逆位转置并不能得到明文；直接执行 LFSR 异或却能同时恢复规范 flag 并命中官方摘要。因此，现有 `data.enc` 对应的是仅经过 `ec` 的数据状态，或者公开汇编并非生成该附件的最终版本。这个结论来自附件实测，不能用源码表面调用顺序覆盖。

## 方法总结

逆向密码工件时应把每个变换独立实现，并分别验证中间结果。源码与附件发生冲突时，可打印明文只是线索，官方摘要才是强校验；本题也说明不能因为函数出现在控制流中，就未经验证地宣称公开密文一定经历了该函数。
