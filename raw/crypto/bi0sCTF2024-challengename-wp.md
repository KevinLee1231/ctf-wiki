# bi0sCTF 2024 - challengename

## 题目简述

服务使用一条参数被隐藏的椭圆曲线完成 ECDSA 签名，并把 flag 编码成曲线点 $F$，再给出公钥点 $Q=dG$ 与密文点 $C=dF$。选手只能请求两次签名；每次签名的消息摘要为 MD5，而 nonce 先与服务端未知的 `magic` 拼接，再经过 MD5 派生。

题目的漏洞链由三部分组成：从已知点恢复曲线参数、借助 MD5 选择前缀碰撞制造 nonce 重用、由两份 ECDSA 签名恢复私钥。得到 $d$ 后即可计算 $F=d^{-1}C$ 并从横坐标还原 flag。

## 解题过程

### 恢复椭圆曲线

设曲线为

$$
y^2=x^3+ax+b\pmod p.
$$

模数 $p$ 已知，公钥点和密文点都在同一条曲线上。任取两个已知点 $(x_1,y_1)$、$(x_2,y_2)$，消去 $b$ 可得

$$
a=\bigl(y_1^2-y_2^2-x_1^3+x_2^3\bigr)(x_1-x_2)^{-1}\pmod p,
$$

随后计算

$$
b=y_1^2-x_1^3-ax_1\pmod p.
$$

在 Sage 中用恢复出的 $a,b$ 建立曲线，并计算曲线群阶 $n$。后续 ECDSA 等式都应在模 $n$ 下求解，而不是模底层有限域 $p$。

### 制造相同 nonce

服务端对用户提交的数据做 `MD5(user_data || magic)`。未知 `magic` 位于消息末尾，因此可以使用一对 MD5 选择前缀碰撞块：两段不同的输入经过各自的碰撞块后具有相同的 MD5 链状态。再在两段消息后附加完全相同的填充数据，MD5 的 Merkle-Damgard 迭代会继续保持碰撞；服务端追加相同的未知 `magic` 后，最终摘要仍相同。

官方 solver 使用两段公开的 MD5 碰撞块，并给二者附加相同的 16 个零字节。服务端虽然拒绝完全相同的 nonce 原文，但这两份原文不同，派生出的 nonce 却相同，因而绕过了检查。

### 从重复 nonce 恢复私钥

两份签名共享同一个临时随机数 $k$，所以其 $r$ 相同，并满足

$$
s_1=k^{-1}(h_1+rd)\pmod n,
$$

$$
s_2=k^{-1}(h_2+rd)\pmod n.
$$

相减后可直接求出

$$
k=(h_1-h_2)(s_1-s_2)^{-1}\pmod n,
$$

再由

$$
d=(s_1k-h_1)r^{-1}\pmod n
$$

恢复私钥。实现时应使用线性同余求解而不是盲目调用逆元：如果某个系数与群阶不互素，先按最大公约数约简并枚举同余类，再用 $dG=Q$ 验证候选。

核心流程可写成：

```python
# h1、h2 是两条被签名消息的 MD5 整数值
k_candidates = solve_linear_congruence(s1 - s2, h1 - h2, n)
for k in k_candidates:
    for d in solve_linear_congruence(r, s1 * k - h1, n):
        if d * G == Q:
            private_key = d
            break

flag_point = inverse_mod(private_key, n) * encrypted_point
flag = int(flag_point[0]).to_bytes((int(flag_point[0]).bit_length() + 7) // 8, "big")
```

最后用 $d^{-1}C=d^{-1}(dF)=F$ 解出明文点。题目把 flag 整数放在 $F$ 的横坐标中，因此将横坐标按大端整数转回字节即可。

## 方法总结

本题的关键不是单独攻击椭圆曲线，而是把多个实现细节串起来：已知曲线点足以恢复短 Weierstrass 曲线参数；MD5 选择前缀碰撞在追加相同未知后缀时仍然成立；ECDSA 一旦重用 nonce，私钥就能由两份签名线性恢复。求解时还应使用公钥关系验证候选，避免非互素同余带来的歧义。
