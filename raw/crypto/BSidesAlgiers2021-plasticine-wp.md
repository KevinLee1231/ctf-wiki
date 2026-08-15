# Plasticine

## 题目简述

服务使用 TFHE 的 TRLWE 密文加密字符串系数。参数为 $Q=2^{64}$、明文模数 $P=2^{10}$、多项式阶数 $N=2^{10}$，同一把秘密密钥同时服务于三个接口：取得加密 flag、加密任意系数向量和解密提交的密文。密钥每 15 秒轮换一次。

解密接口会把前若干系数转成字符串；如果结果恰好等于真实 flag，就返回玩具 flag。这个检查只比较解密后的原文，没有验证提交的密文是否就是服务器给出的 flag 密文，也没有阻止 TRLWE 的同态加法。

## 解题过程

TRLWE 密文可以写成 $(\mathbf a,b)$，其中

$$
b=\langle\mathbf a,\mathbf s\rangle+m+e \pmod Q.
$$

实现中的整数 $x\in[0,P)$ 会被编码为 $x\cdot Q/P$。因此，在密文的 $b$ 多项式相应系数上增加

$$
\Delta\cdot Q/P
$$

会使解密结果对应增加 $\Delta\bmod P$，而掩码 $\mathbf a$ 无须变化。

官方 solver 取 $\Delta=10$，但把长度硬编码为 40；仓库登记的 flag 实际有 45 个字符，直接照抄会截断。更稳妥的写法是修改整个 $b$ 多项式：服务器检查的 flag 前缀必然被改变，而原文长度可以在还原后通过首个零系数确定。

~~~python
import requests

Q = 2**64
P = 2**10
DELTA = 10

ciphertext = requests.get(f"{BASE_URL}/encryptedFlag").json()

for i in range(len(ciphertext["b"]["coefficients"])):
    value = ciphertext["b"]["coefficients"][i]
    ciphertext["b"]["coefficients"][i] = (
        value + DELTA * (Q // P)
    ) % Q

coeffs = requests.post(
    f"{BASE_URL}/decrypt",
    json=ciphertext,
).json()

plain = [(value - DELTA) % P for value in coeffs]
flag_length = plain.index(0)
flag = "".join(chr(value) for value in plain[:flag_length])
print(flag)
~~~

取得密文、修改并提交解密必须落在同一个 15 秒密钥周期内，否则新的秘密密钥无法正确解开旧密文。恢复结果为：

~~~text
shellmates{rlw3_1s_fun__y0u_c4n_p14y_w1th_1t}
~~~

## 方法总结

本题并不是破解 RLWE 私钥，而是利用可塑性绕过“禁止解密某条明文”的业务检查。看到同态密文和解密 oracle 共用密钥、服务只按解密结果做黑名单比较时，应优先尝试在密文域施加可逆偏移。还要同步检查密钥轮换周期；代数关系正确但跨过轮换时间，同样会表现为随机解密失败。
