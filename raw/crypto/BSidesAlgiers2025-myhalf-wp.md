# BSidesAlgiers2025 - MyHalf

## 题目简述

题目给定 RSA 参数与部分私钥信息：

- 模数 `N`
- 指数 `e=0xef1b`
- 密文 `c`
- 私钥 `d_p`、`d_q` 的低位 512bit（`LSB_dp`、`LSB_dq`）

目标是利用这条偏私钥信息恢复完整 `d_p`、`d_q`，进而分解 `N`、求出私钥 `d`，解密得到 flag。

## 解题过程

### 1. 建模与已知量

对于 RSA，有

$$
e d_p - 1 = k(p-1),\quad e d_q - 1 = \ell(q-1),\quad N = p q
$$

`cipher.txt` 给出了 `N,e,c`，并公开 `d_p,d_q` 的低 512 位：

$$
d_p = d_{p,\text{MSB}} \cdot 2^{512} + \mathrm{LSB}_{d_p}
$$

同理可写 `d_q`。已知位数下，未知 MSB 比例约为每个私钥参数剩余位数。

### 2. 先恢复 $k,\ell$

`solve.sage` 的 `recover_k_l_brutal_modular` 基于同源论文推导（代码注释标 `Equation (8)`）先消掉高位变量，得到可解线性同余：

$$
M = e \cdot 2^{512}
$$

遍历 $k \in [1,e)$，解

$$
(k(N-1) - (e\mathrm{LSB}_{d_p}-1))\cdot \ell \equiv k(e\mathrm{LSB}_{d_q}-1)-A \pmod M
$$

其中

$$
A = -e^2\mathrm{LSB}_{d_p}\mathrm{LSB}_{d_q}+e\mathrm{LSB}_{d_p}+e\mathrm{LSB}_{d_q}-1 \pmod M
$$

`e` 只有 16 bit，所以该阶段很快可穷举到真实 $(k,\ell)$。

### 3. 恢复 $d_p$ 的 MSB（格化）

在已得 $k$ 后，脚本构造一元多项式并做 LLL：

$$
f(x)=x+\mathrm{IN}_k\cdot(e\cdot \mathrm{LSB}_{d_p}-1+k)
$$

其中 $\mathrm{IN}_k = (e2^{512})^{-1} \bmod (kN)$。

通过移位/拼接 `f^i*k^{m-i}*N^{\max(0,t-i)}` 构造格，取 LLL 短向量恢复候选多项式，再提取 $[0,2^{\text{UnknownMSB}})$ 的整数根。脚本内置：
- `recover_msb_dp_with_lll`
- `m2=40, t2=20`（经验参数，可调）

得到 $d_p$ 的高位后，重组完整私钥部分：

$$
\hat d_p = \text{msb}\cdot 2^{512} + \mathrm{LSB}_{d_p}
$$

### 4. 因式分解与解密

脚本按标准 RSA 关系验证：

$$
g=\gcd(e\hat d_p - 1 + k,\ N)
$$

若 $1<g<N$，则 $g=p$，进而 $q=N/p$。再算

$$
\phi(N)=(p-1)(q-1),\quad d=e^{-1}\bmod\phi(N),\quad m=c^d \bmod N
$$

得到明文 flag 字节。这里的 `+k` 正是由 $e d_p-1=k(p-1)$ 移项得到；若误写成 $kp$，后续 `gcd` 公式就无法自洽。

### 5. 可执行链条

```bash
# 先在本题目录执行
sage solve/solve.sage
```

`cipher.txt` 中已给出 solver 所需的全部参数，脚本会打印中间恢复结果与解密明文。本轮没有重新运行参数为 `m2=40,t2=20` 的 LLL 阶段；仓库也没有保存明文 flag，定向检索未找到可独立核验的公开最终输出。因此本文只归档官方 solver 所定义的完整恢复链，不虚构最终 flag，也不把它表述为本地实测通过。

## 方法总结

- 核心技巧：先将“已知低位+小公钥指数”的约束转成 $k,\ell$ 的线性同余，再用格降维从 `lsb` 上恢复 `d_p` 的 MSB。
- 识别信号：当 `e` 很小且给出 `dp/dq` 下半段时，优先考虑 `gcd` 回推法 + Coppersmith/LLL 的组合。
- 复用要点：这类题模板化很强，关键是参数关系式要对齐（尤其 `M=e*2^L` 的同余约束和未知位边界）而非堆参数搜索。
