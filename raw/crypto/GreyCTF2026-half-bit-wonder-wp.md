# Half Bit Wonder

## 题目简述

题目给出一个经过 LadderLeak/Bleichenbacher 风格预处理的 Hidden Number Problem。模数 $q$ 是 secp256k1 标量阶，公开数据分成 $T=20$ 个区块。第 $t$ 个区块含公开乘子 $r_t$ 和 4096 组 $(a_i,z_i)$，其中 $1\le a_i<L$、$L=2^{20}$，隐藏值满足：

$$
y_t=r_t\cdot sk\bmod q
$$

$$
k_i=z_i+a_i y_t\bmod q,\qquad 0\le k_i<q/2
$$

题目已经替选手完成了真实侧信道采集与碰撞降维，攻击目标是利用 nonce 只落在模数下半区这一点，从多个区块恢复共同私钥 $sk$。

## 解题过程

对于正确的候选 $w=y_t$，角度

$$
\theta_i(w)=2\pi\frac{z_i+a_iw}{q}
$$

只覆盖单位圆的下半个模区间，对应复向量不会完全抵消；错误 $w$ 使相位近似均匀。于是定义偏差分数：

$$
B_t(w)=\left|\sum_i\exp\left(2\pi\mathrm{i}\frac{z_i+a_iw}{q}\right)\right|
$$

由于所有 $a_i<L$，可在长度 $L$ 的数组中按系数聚合：

$$
A[a_i]\mathrel{+}=\exp\left(2\pi\mathrm{i}\frac{z_i}{q}\right)
$$

计算 `ifft(A) * L`，一次 FFT 就能同时评估网格 $w_j\approx jq/L$ 上的全部偏差分数。每个区块保留绝对值最大的若干峰，并把峰编号换算为模 $q$ 的区间中心：

```python
A = np.zeros(L, dtype=np.complex128)
for a, z in block["samples"]:
    A[int(a)] += cmath.exp(2j * np.pi * float(int(z)) / float(q))

scores = np.abs(np.fft.ifft(A) * L)
j = int(np.argmax(scores))
center = ((j * q) + L // 2) // L % q
```

单个区块只能把 $y_t$ 定位到宽约 $q/L$ 的区间，仍有约 236 位不确定性，不能局部爆破。需要把 20 个粗略观测作为同一个私钥的多维近似关系：

$$
r_tsk\approx c_t\pmod q
$$

构造行格：

$$
\Lambda=\left\{
(r_0x+qm_0,\ldots,r_{T-1}x+qm_{T-1})\mid x,m_i\in\mathbb Z
\right\}
$$

它由向量 $(r_0,\ldots,r_{T-1})$ 和 $q e_0,\ldots,q e_{T-1}$ 生成。真实 $x=sk$ 对应的格向量与 FFT 峰中心向量 $c=(c_0,\ldots,c_{T-1})$ 很接近，因此用 Sage 的 `IntegerLattice.approximate_closest_vector` 求 CVP 即可。

从所得向量 $v$ 的每一维反推出候选私钥：

$$
sk_t=v_t r_t^{-1}\bmod q
$$

只有全部 20 个 $sk_t$ 一致时才接受。若各区块的第一峰组合失败，官方脚本会在峰值间隔最弱的少数区块中逐一替换为第二至第四峰，而不是指数枚举所有组合。

最后以私钥十进制字符串派生 SHAKE256 密钥流：

```python
stream = hashlib.shake_256(
    ("nothing to see here:" + str(sk)).encode()
).digest(len(ciphertext))
plaintext = bytes(a ^ b for a, b in zip(ciphertext, stream))
```

得到：

```text
grey{bLe1cH3nb4Ch3R_iZ_n0_1_Hit_WonD3R_oK4y???}
```

## 方法总结

“只泄露半个比特”不等于信息不足：大量 nonce 同时落在 $[0,q/2)$ 会形成可统计检测的圆周偏差。FFT 把每区块 $2^{20}$ 个候选的评分从逐点计算降为一次卷积式求值；CVP 再把多个极粗糙的模乘近似合并为唯一私钥。两层方法各解决一个尺度问题，缺少任一层都无法从 256 位模数中完成恢复。
