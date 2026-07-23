# Ekans

## 题目简述

题目同样采用 LWE 形式

$$
B=As+e\pmod q,
$$

但生成器为了“节省随机性”，没有独立生成误差 $e$，而是直接把秘密向量 $s$ 逆序：

```python
s = gen_error_vec()
e = matrix(s.tolist()[::-1])
B = (A * s + e) % q
```

这个确定性关系把本应隐藏秘密的噪声变成了秘密的已知线性变换，使 LWE 方程退化为普通线性方程。

## 解题过程

构造反序置换矩阵 $F$，其副对角线为 1，其余位置为 0，于是

$$
e=Fs.
$$

代入公开密钥方程：

$$
B=As+Fs=(A+F)s\pmod q.
$$

在模 $q=15901$ 的环上直接解线性系统即可恢复秘密：

```python
flip = zero_matrix(ZZ, n, n)
for i in range(n):
    flip[i, n - i - 1] = 1

R = IntegerModRing(q)
s = (A + flip).change_ring(R).solve_right(B)
```

加密端给出

$$
B'=S'A+E',\qquad
V=S'B+e'+\operatorname{Encode}(m).
$$

有了 $s$ 后计算 $V-B's$，噪声只会让结果在每个编码中心附近小幅波动。题目按十六进制字符编码，每级间隔 $q/16$，所以将每个分量乘以
$16/q$ 后四舍五入、模 16，再把十六进制串转回字节：

```python
decoded = ""
for value in (V - B_prime * s) % q:
    nibble = round(int(value[0]) * 16 / q) % 16
    decoded += "0123456789abcdef"[nibble]

print(bytes.fromhex(decoded).decode())
```

最终得到：

```text
UMDCTF{w3ll_m@ybE_1_5h0uLdnt_h4v3_tRi3d_t0_s@ve_0N_thE_raNdomn3ss._Le55on_l34rn3d}
```

## 方法总结

- 核心技巧：把“误差等于秘密逆序”写成置换矩阵关系，将 LWE 直接化为模线性方程。
- 识别信号：噪声、nonce 或随机量若由秘密通过公开确定性函数生成，应先检查能否移项合并，而不是立即使用通用格攻击。
- 复用要点：恢复秘密后仍要完全复现消息量化规则；本题每个矩阵元素代表一个十六进制半字节，而不是一个完整字符。
