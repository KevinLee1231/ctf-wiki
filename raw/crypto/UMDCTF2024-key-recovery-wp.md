# Key Recovery

## 题目简述

题目使用 RSA 公钥和 PKCS#1 OAEP 加密 flag，但给出的私钥 PEM 被逐行截断。生成脚本对 PEM 的第 $i$ 行只保留前 $66-2i$ 个字节，因此 Base64 数据形成阶梯状缺失；与此同时还给出了加密脚本、损坏的 `modified.pem` 和 OAEP 密文。

这不是直接分解 RSA 模数的问题。PEM 中仍泄露了 $n,e,d,p,q,d_p,d_q$ 的大量连续比特，可以利用 RSA 各参数之间的代数关系逐段补回，最后只对一小段未知量使用 LLL。

## 解题过程

### 从损坏 Base64 建立已知位掩码

将每一行右侧分别用 Base64 字符 `A` 和 `/` 补到 64 字符，再解码成两份 DER 数据。`A` 对应六比特全零，`/` 对应六比特全一，因此两份结果的异或会准确标出原 PEM 中缺失的比特：

```python
lines_0 = [line.ljust(64, "A") for line in lines]
lines_1 = [line.ljust(64, "/") for line in lines]

known_0 = base64.b64decode("".join(lines_0))
known_1 = base64.b64decode("".join(lines_1))
missing_mask = bytes_to_long(known_0) ^ bytes_to_long(known_1)
```

官方脚本依据该 DER 的固定字段位置取出各参数，并把每段连续未知位记录成
`[start, length]`：

```python
n,  n_missing  = get_region(38,  256)
e,  e_missing  = get_region(296,   3)
d,  d_missing  = get_region(303, 256)
p,  p_missing  = get_region(563, 128)
q,  q_missing  = get_region(695, 128)
dp, dp_missing = get_region(827, 128)
dq, dq_missing = get_region(959, 128)
```

这里的长度单位为字节；实际未知区域由掩码进一步细分。用两个极端字符补位的目的只是区分已知位和未知位，不能把任一补位结果直接当成私钥。

### 用 RSA 参数关系逐段恢复

设

$$
ed-1=k\lambda(n),\qquad
\lambda(n)=\frac{(p-1)(q-1)}{\gcd(p-1,q-1)}.
$$

官方脚本由现有高位估计得到 $\gcd(p-1,q-1)=10$，并近似计算
$\lambda(n)$ 与整数 $k$。随后交替使用

$$
n=pq,\qquad
\varphi(n)=n-p-q+1,\qquad
10(ed-1)=k\varphi(n)
$$

修复 $n$ 和 $d$ 的高位。

对低位则在模 $2^t$ 下求解。例如已知 $p,e,d$ 时，

$$
\varphi(n)\equiv 10(ed-1)k^{-1}\pmod{2^t},
$$

再结合 $pq\equiv n\pmod{2^t}$ 可逐步恢复 $q$ 和 $n$。遇到 $p-1$、$k_p$ 或常数 10 不可逆时，先除去与 $2^t$ 的最大公因子，再在缩小后的模数上求逆。

CRT 指数还提供了额外约束：

$$
ed_p-1=k_p(p-1),\qquad 0<k_p<e.
$$

由已知位可以先估计 $k_p$，再交替补回 $d_p$ 与 $p$。当已知 $n$ 和
$p+q$ 的低位时，脚本还在模 $2^t$ 下求二次方程

$$
X^2-(p+q)X+n\equiv0\pmod{2^t}
$$

来同时扩展 $p,q$ 的已知区域。

### 用 LLL 补最后一段未知比特

在代数传播后，$p$ 和 $q$ 各只剩一段较短的中间未知量。写成

$$
p'=p+\varepsilon_p2^s,\qquad
q'=q+\varepsilon_q2^s,
$$

并利用 $p'q'\equiv n\pmod{2^t}$，忽略在当前模数下消失的高次项，可得到关于
$\varepsilon_p,\varepsilon_q$ 的线性模关系。官方脚本把它嵌入一个带尺度因子的四维整数格：

```sage
L = matrix(ZZ, [
    [0, Wp, 0, S],
    [0, 0, Wq, (p * inverse_mod(q, M) % M) * S],
    [S, 0, 0, ((p*q - n) // start * inverse_mod(q, M) % M) * S],
    [0, 0, 0, M * S],
])
```

LLL 中末坐标为零的短向量给出两个误差项，填回后即可继续用前述等式恢复完整
$n,d$。

### OAEP 解密

得到一致的 $(n,e,d)$ 后直接构造 RSA 私钥，并使用与加密端相同的
PKCS#1 OAEP 解密：

```python
key = RSA.construct((int(n), int(e), int(d)))
cipher = PKCS1_OAEP.new(key)
print(cipher.decrypt(ciphertext))
```

最终得到：

```text
UMDCTF{impressive_recovery!_i_forgot_to_tell_you_this_but_the_private_key_ends_with_VATE KEY-----}
```

## 方法总结

- 核心技巧：把截断 PEM 转成 RSA 参数的已知位/未知位模型，用 $n=pq$、$ed-1=k\lambda(n)$ 和 CRT 指数关系传播比特，最后以低维 LLL 收尾。
- 识别信号：私钥文件虽然损坏，但 DER 字段边界仍可定位，且多个 RSA 私钥参数同时部分泄露时，不应只盯着模数分解。
- 复用要点：Base64 补位只用于构造掩码；在模 $2^t$ 下除法前必须处理偶因子，否则错误地求逆会直接破坏后续所有已知位。
