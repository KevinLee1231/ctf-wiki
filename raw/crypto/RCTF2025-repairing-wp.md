# RePairing

## 题目简述

服务在 BLS12-381 配对群上实现了一个 ElGamal 风格的加密方案。启动时给出公钥、身份散列、挑战密文三元组和被异或加密的 flag；随后允许解密任意一个“不与原三元组完全相等”的密文，并返回解密共享秘密经 KDF 派生的字节流。方案本身支持公开重随机化，而服务端只做字节级去重，因此可以把挑战密文变成另一个合法表示，再向解密 oracle 查询同一明文。

## 解题过程

### 写出挑战密文

令：

```text
G  = G1 的生成元
q  = H1(id)，位于 G2
pk = 公钥，位于 GT
M  = 随机共享秘密，位于 GT
t  = 加密随机标量
```

源码的加密函数为：

```text
c1 = M * pk^t        (GT 中乘法)
c2 = G * t           (G1 中标量乘法)
c3 = q * t           (G2 中标量乘法)
```

flag 并不直接放在该密文中，而是：

```text
encrypted_flag = flag xor KDF(M, len(flag))
```

服务 banner 依次给出 `id | DST | pk | q | c1 | c2 | c3 | encrypted_flag`，其中各群元素使用 arkworks 的压缩序列化再十六进制编码。

### 把随机数从 `t` 变成 `t+r`

任选非零标量 `r`，用公开量构造：

```text
c1' = c1 * pk^r
c2' = c2 + G*r
c3' = c3 + q*r
```

代入原式：

```text
c1' = M * pk^(t+r)
c2' = G * (t+r)
c3' = q * (t+r)
```

这不是伪造一个“近似”密文，而是明文仍为 `M`、随机数改为 `t+r` 的另一份完全合法密文。配对解密中的分子、分母会像正常密文一样消去与 `t+r` 有关的项，结果仍是 `M`。

### 绕过去重并恢复 flag

服务端只拒绝：

```rust
if c1_q == ct.c1 && c2_q == ct.c2 && c3_q == ct.c3 {
    println!("no");
    return;
}
```

它还检查 `c2'`、`c3'` 非零且位于正确子群，但重随机化后的合法群元素天然通过这些检查。实现时按 banner 中的曲线参数反序列化元素，然后执行：

```rust
let r = random_non_zero_scalar();
let c1_new = c1 * pk.pow(r.into_bigint());
let c2_new = c2 + G1Projective::generator() * r;
let c3_new = c3 + q * r;
```

把三个新元素按与服务相同的压缩格式发送。解密 oracle 对它们恢复 `M`，返回的正是原挑战所用的：

```text
key = KDF(M, len(encrypted_flag))
```

最后逐字节异或即可：

```python
flag = bytes(a ^ b for a, b in zip(encrypted_flag, returned_key))
```

这里不要尝试修改单个分量：三个分量共享同一个随机标量，必须同时把 `t` 平移为 `t+r`，否则密文关系被破坏，解密结果也不再是原共享秘密。

## 方法总结

漏洞是“可重随机化密码方案”和“只禁止原始字节串的解密 oracle”组合造成的。服务端验证了曲线点与子群，却没有验证查询密文是否与挑战密文属于同一个等价类。遇到 ElGamal、配对加密或同态密文时，应先检查能否用公开量把随机数平移而保持明文不变；防御上必须从协议层禁止挑战明文的等价密文查询，不能只比较序列化后的三元组是否完全相同。
