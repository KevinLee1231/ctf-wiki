# Noisy

## 题目简述

附件生成 20 个样本，模数为 $n=pq$，其中 $p$ 为 512-bit 素数、$q$ 为 1024-bit 素数。每个公开值是

$$
c_i=s\,(m_i+k_{0,i}M)(1+k_{1,i}p)\pmod n,
$$

其中 $m_i<2^{24}$、$M,k_{0,i}<2^{30}$、$k_{1,i}<2^{512}$，而 $s,p,M$ 均未公开。flag 并不直接出现在样本中：题目以 `md5(str(msg).encode())` 为 16-byte AES-ECB key，加密 PKCS#7 padding 后的 flag。因此主线是恢复完整、有序的 `msg` 列表，而不是分解 AES。

## 解题过程

### 第一层：从乘法噪声恢复近似公因子样本

令 $x_i=m_i+k_{0,i}M$，则 $x_i<2^{60}+2^{24}$。模 $p$ 后，扰动项与公共乘子只留下比例关系：

$$
c_i\equiv s x_i\pmod p.
$$

选最后一个样本作归一化，构造

$$
v_i\equiv-c_i c_{19}^{-1}\pmod n\quad(0\le i<19),
$$

以及行格

$$
L=\begin{pmatrix}I_{19}&v\\0&n\end{pmatrix}.
$$

对 $L$ 做 LLL，去除末尾明显较长的基向量；这些短关系与 $(x_0,\ldots,x_{19})$ 正交。再取短关系矩阵的右核，可恢复与 $x$ 成比例的候选向量 $r$。[同届公开复现](https://mi1n9.github.io/2025/05/11/2025miniLCTF/) 使用的关键步骤是 `Matrix(short_rows).right_kernel()`：LLL 输出的是正交关系，不能误把它本身当成 $x$。

### 第二层：由 $m_i+k_{0,i}M$ 恢复消息

$r$ 仍带有 $x_i=m_i+k_{0,i}M$ 的结构，所有 $m_i$ 只有 24 bit。以 $\alpha=2^{24}$，对 $r$ 构造第二个正交格：将单位阵与列向量 $\alpha r$ 拼接，LLL 后再次取保留短行的右核并约简。最短向量给出候选 `msg`。

这一步相当于近似公因子问题：各 $x_i$ 到共同倍数 $k_{0,i}M$ 的偏差同时很小。恢复后至少检查：

- `len(msg) == 20`，且每项在 $[0,2^{24})$；
- 使用 Python 列表的默认字符串表示计算 `md5(str(msg).encode())`，不要改成拼接二进制整数；
- AES 解密结果的 PKCS#7 padding 正确，并符合 flag 格式。

### 生成 key 并解密

解法的最后部分无需恢复 $s,p,M,k_0,k_1$：

```python
key = md5(str(msg).encode()).digest()
pt = AES.new(key, AES.MODE_ECB).decrypt(bytes.fromhex(encrypted_flag))
flag = unpad(pt, 16)
```

`msg` 的顺序就是输出样本的顺序，不能将第二个格的向量排序、取绝对值或自行规范化后再哈希。padding 与明文格式共同作为格候选的最终验证。

## 方法总结

- 核心技巧：把模 $p$ 后只差公共比例的 noisy samples 写成正交格，先恢复短 $x_i$，再用第二个 AGCD 格恢复 24-bit 消息。
- 识别信号：表达式含有 $(1+k p)$、多个样本共用 $p$ 和很短的内层量时，先检查模 $p$ 是否使噪声消失；不必先硬分解 $n$。
- 复用要点：正交格需通过右核回收目标向量；列表序列化、ECB 模式和 PKCS#7 都是题目定义的一部分，不能只凭“可读输出”接受候选。
