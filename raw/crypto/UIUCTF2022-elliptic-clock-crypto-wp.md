# Elliptic Clock Crypto

## 题目简述

题目仿照椭圆曲线 Diffie-Hellman 编写了一套“时钟曲线”密码系统。点满足

$$
x^2+y^2\equiv1\pmod p,
$$

加法定义为

$$
(x_1,y_1)+(x_2,y_2)=
(x_1y_2+y_1x_2,\ y_1y_2-x_1x_2).
$$

双方分别公开 $A=aG$、$B=bG$，共享点为 $abG$。程序把共享点的字符串表示做 MD5，得到 16 字节 AES-ECB 密钥并加密 flag。问题不在 AES，而在自定义群的阶是平滑的，离散对数可以通过 Pohlig–Hellman 快速求解。

## 解题过程

### 确认群结构

单位元是 $(0,1)$，点 $(x,y)$ 的逆元为 $(-x,y)$。把群运算代回约束可验证结果仍满足 $x^2+y^2=1$，因此这些点在给定运算下形成有限群。对题目选用的素数，官方分析得到群阶为

$$
|C(p)|=p-1.
$$

真正的薄弱点是 $p-1$ 可以轻易分解为小素数幂乘积，而不是曲线方程看起来是否像标准 ECC。

### Pohlig–Hellman 恢复私钥

设 $A=aG$。将

$$
p-1=\prod_i r_i^{e_i}
$$

分解后，对每个素数幂 $m_i=r_i^{e_i}$ 计算

$$
G_i=\frac{p-1}{m_i}G,\qquad
A_i=\frac{p-1}{m_i}A.
$$

于是只需在阶为 $m_i$ 的子群中求解

$$
A_i=a_iG_i,
$$

官方脚本对每个子群使用 BSGS，最后通过中国剩余定理合并 $a_i$：

```python
residues = []
moduli = []

for r, e_ in factor(p - 1):
    modulus = int(r**e_)
    cofactor = (p - 1) // modulus
    Gi = scalar_mult(base_point, cofactor)
    Ai = scalar_mult(alice_point, cofactor)
    ai = bsgs(Gi, Ai, modulus)
    residues.append(int(ai))
    moduli.append(modulus)

alice_secret = crt(residues, moduli)
assert scalar_mult(base_point, alice_secret) == alice_point
```

这里的验证非常重要：CRT 得到的指数必须重新乘基点并与 Alice 公钥完全一致，才能排除子群离散对数或模数处理错误。

### 还原 AES 密钥

恢复 $a$ 后，计算共享点 $S=aB$。题目使用的是点字符串的 MD5，而不是坐标拼接后的二进制编码，因此密钥生成必须严格复刻：

```python
shared = scalar_mult(bob_point, int(alice_secret))
key = md5(f"({shared[0]},{shared[1]})".encode()).digest()
flag = AES.new(key, AES.MODE_ECB).decrypt(flag_enc)
```

对 `output.txt` 中的两组公钥和密文执行该流程，得到：

```text
uiuctf{Circle5_ar3_n0t_ell1ptic}
```

## 方法总结

- 核心技巧：识别自定义单位圆群的阶，并利用平滑群阶上的 Pohlig–Hellman 与 BSGS 求离散对数。
- 识别信号：题目声称使用 ECC，但点集、群运算和参数都是自定义的；安全性应由基点阶及其最大素因子决定，不能由坐标位数判断。
- 复用要点：先验证封闭性、单位元、逆元和实际基点阶，再分解群阶；恢复私钥后要复算公钥，并精确复刻共享点序列化方式，避免在最后的 KDF 步骤得到错误密钥。
