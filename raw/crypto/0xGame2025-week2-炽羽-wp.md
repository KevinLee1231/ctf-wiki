# 炽羽

## 题目简述

题目把一个各分量均不超过 255 的短向量 `secret` 与若干约 $2^{256}$ 量级的长向量组成格基，再用随机整数线性组合打乱基。服务端给出打乱后的 $4\times4$ 整数矩阵；恢复 `secret` 后提交其十六进制形式，服务端返回一段缺失原始 IV 的 AES-CFB 密文。

## 解题过程

### 用 LLL 恢复短向量

整数行变换只更换格基，不改变这些向量张成的格。原基中 `secret` 远短于其它方向，因此格内最短方向高度突出；LLL 规约后，首个短向量通常为：

原 PDF 附有 [CTF Wiki 的格概述](https://ctf-wiki.org/crypto/asymmetric/lattice/overview/) 和 [格与 LLL 基础讲义](https://crystaljiang232.github.io/crypto/lattice/)。本题真正使用到的条件是“整数可逆基变换不改变格”以及“LLL 倾向输出较短、近似正交的基”，这些条件已在正文结合具体量级说明。

$$
v=\pm k\cdot secret
$$

格对取负封闭，所以规约结果可能是负向量；随机基变换也可能留下整数倍。对各分量取绝对值，再除以它们的最大公约数即可规范化出原字节向量。该方法依赖明显的长度差：若其它基向量也很短，LLL 不再保证首向量沿 `secret` 方向。

### 从后续密文恢复明文

原稿将这里写成 OFB，但求解脚本和反馈关系实际对应 AES-CFB。CFB 的下一段加密状态由前一段密文反馈；即使最初 IV 丢失，只要已知前一个完整密文块，就能把它作为新 IV，从下一块开始同步解密。因此以 `ciphertext[:16]` 为 IV，解密 `ciphertext[16:]`，只能损失第一块明文。

```sage
from sage.all import matrix, ZZ, gcd
from pwn import *
from Crypto.Cipher import AES

io = process(['python', 'task.py'])
io.recvuntil(b':')
basis = matrix(ZZ, 4, 4)
for i in range(4):
    io.recvuntil(b'[')
    basis[i] = list(map(int, io.recvuntil(b']', drop=True).decode().split()))

reduced = basis.LLL()
short = reduced[0]
scale = gcd(list(short))
secret = bytes(abs(int(x)) // abs(int(scale)) for x in short)
io.sendlineafter(b':', secret.hex().encode())

io.recvuntil(b':')
ciphertext = bytes.fromhex(io.recvline().decode().strip())
key = b'0xGame2025awaQAQ'
cipher = AES.new(key, AES.MODE_CFB, iv=ciphertext[:16])
print(cipher.decrypt(ciphertext[16:]))
```

## 方法总结

- 核心技巧：利用格中异常短的隐藏向量做 LLL 规约，并利用 CFB 的密文反馈从第二块重新同步。
- 识别信号：短字节向量与超长随机向量被整数线性组合；密文丢失原始 IV 但保留连续前置密文块。
- 复用要点：LLL 结果允许符号和整数倍差异，需要规范化；反馈模式必须以源码中的实际 `MODE_CFB` 为准，不能只凭文字标签判断。
