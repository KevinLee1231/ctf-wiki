# DownUnderCTF 2022 kyber± Writeup

## 题目简述

题目使用 Kyber512，工作环为

$$
R_q=\mathbb Z_{3329}[X]/(X^{256}+1),
$$

秘密是两个多项式组成的向量 $\mathbf s=(s_0,s_1)$。服务公开公钥、秘密钥末尾的 64 字节，并提供 7681 次 encapsulation/decapsulation 调用；结束后输出 `flag XOR serialized_secret_key`。

Kyber 的 CPA 层密文为 $(\mathbf u,v)$，解密近似计算：

$$
v-\mathbf s\cdot\mathbf u=E+\mu,
$$

其中 $E$ 是小误差多项式，$\mu$ 的每个系数由消息位编码为 0 或 1665。题目补丁取消了 $v$ 的 4 位压缩，使攻击者能逐单位修改系数；同时破坏了 KEM 重加密失败时的常数时间条件移动。

## 解题过程

### 从补丁构造“消息是否改变”oracle

标准 Kyber decapsulation 会解密得到 $m'$，用它重新加密出 $c'$；若 $c\ne c'$，就用秘密随机值 $z$ 替换 pre-key，返回

$$
\operatorname{KDF}(z\parallel H(c)).
$$

原版 `verify` 只返回 0/1，`cmov` 再把 1 扩展成 `0xff` 掩码。题目却让 `verify` 返回不同字节数量（最多 `0xff`），并删掉 `cmov` 中的取负。若输入密文只与重加密结果相差少数字节，pre-key 只被部分位覆盖；若解出的消息发生变化，重加密结果几乎完全不同，计数饱和为 `0xff`，pre-key 才会完整换成 $z$。

服务所谓 `H(pk)` 实际打印了秘密钥最后 64 字节，即 $H(pk)\parallel z$，所以 $z$ 已知。收到 shared secret 后即可判断它是否等于完整失败分支：

```python
def message_changed(z, ciphertext):
    shared = decapsulate(ciphertext)
    fallback = shake_256(z + sha3_256(ciphertext).digest()).digest(32)
    return shared == fallback
```

### 逐系数恢复误差多项式

本地用公开密钥生成一个合法密文 $(\mathbf u,v)$，同时保留其 32 字节消息，从而知道每个消息位 $b_i$。对 $v$ 的第 $i$ 个系数增加 $oX^i$，二分搜索使解密消息第一次改变的最小 $o_i$。

Kyber 的 1 位解码在系数落入 $[833,2496]$ 时输出 1。若 $b_i=0$，第一次翻转满足 $E_i+o_i=833$；若 $b_i=1$，第一次越过另一端满足 $E_i+1665+o_i=2497$。两种情况统一为：

$$
E_i=833-b_i-o_i.
$$

```python
for i in range(256):
    o = first_offset_that_changes_message(i)
    bit = (message[i // 8] >> (i % 8)) & 1
    error[i] = 833 - bit - o
```

对两个独立合法密文执行该过程，二分查询总量可控制在 7681 次以内，得到 $E_1,E_2$。

### 解线性方程恢复秘密向量

计算：

$$
y_i=v_i-E_i-\mu_i.
$$

则在 $R_q$ 中有：

$$
s_0u_{1,0}+s_1u_{1,1}=y_1,
\qquad
s_0u_{2,0}+s_1u_{2,1}=y_2.
$$

消元得到：

$$
s_1=(y_2-y_1u_{1,0}^{-1}u_{2,0})\cdot
(u_{2,1}-u_{1,0}^{-1}u_{1,1}u_{2,0})^{-1},
$$

$$
s_0=(y_1-s_1u_{1,1})u_{1,0}^{-1}.
$$

乘法和求逆都在商环 $R_q$ 中进行；若随机多项式不可逆，可重新生成样本。

恢复的是普通域表示的 $\mathbf s$，而 Kyber 私钥序列化保存的是 NTT 域表示。调用参考实现的 `polyvec_ntt` 后再按 12 位系数格式序列化，最后与 `flag_enc` 异或：

```python
secret_ntt = polyvec_ntt(secret_normal)
secret_bytes = polyvec_to_bytes(secret_ntt)
flag = xor(flag_enc, secret_bytes[:len(flag_enc)])
```

得到：

```text
DUCTF{w3ll_d0n3_0n_und3rst4nd1ng_Kyber_4s_cl34r_4s_crystal..._n0w_c4n_y0u_br34k_1t_w1th0ut_th3_p4tch??_11f0ffee7cfaf83b794a05cc97293dc936f0fcb6164790ab}
```

## 方法总结

本题利用链由三处条件共同成立：服务泄露 $z$，错误的 `verify/cmov` 把 KEM 失败路径变成可识别 oracle，取消 $v$ 压缩又提供了单系数精细扰动能力。先用阈值 oracle 恢复两个误差多项式，再把 Kyber 解密关系化为 $R_q$ 上的二元线性方程即可恢复秘密。实现格密码攻击时还必须区分普通域、NTT 域和序列化格式，否则数学上正确的秘密仍无法解开按字节异或的 flag。
