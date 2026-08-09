# Hadis Paide

## 题目简述

服务把 flag 解释为整数，并用 Paillier 体制解密玩家提交的密文。它不返回明文，只比较本次明文与 flag 的汉明距离是否比上一次更小。玩家另有一次免费加密任意正整数的机会。决定性弱点是 Paillier 的加法同态性与“Warmer/Colder”距离侧信道可以组合使用。

## 解题过程

请求免费加密 $1$，得到 $E(1)$。Paillier 满足

$$
E(a)E(b)\bmod N^2=E(a+b),\qquad E(a)^k\bmod N^2=E(ka).
$$

令 `b = E(1)`、`c = E(1)`，先提交一次 `c` 建立距离基线。随后每轮把 `b` 平方，使其明文依次变为 $2,4,8,\ldots$，再令 `c = c*b mod N^2`。相邻两次候选明文只会翻转当前考察位及其进位关系，因此距离变近说明该位为 `1`，变远说明该位为 `0`。官方 solver 的核心循环如下：

```python
c = enc_one
b = enc_one
play(c)

bits = ""
for _ in range(1024 - 32):
    b = pow(b, 2, N * N)
    c = c * b % (N * N)
    result = play(c)
    bits = ("1" if result.startswith("W") else "0") + bits

flag_int = (int(bits, 2) << 1) ^ 1
print(flag_int.to_bytes(128, "big").lstrip(b"\x00"))
```

恢复结果为：

```text
n00bz{0k4y_h34r_m3_out_c0m1ng_up_w1th_fl4gs_1snt_4lw4ys_34sy_3sp3c14lly_l0ng_fl4gs_4nyw4y_y0u_4r3_d01ng_4w3s0m3_k33p_1t_up}
```

## 方法总结

本题并未直接破坏 Paillier，而是滥用了同态运算和可重复查询的距离反馈。面对只返回“更接近/更远”的接口，不能把它当作无信息布尔值；只要能构造相邻、可控的候选明文，它就可能退化为逐位 oracle。
