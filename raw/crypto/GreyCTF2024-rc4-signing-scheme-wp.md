# rc4-signing-scheme

## 题目简述

所谓签名实际是用 RC4 加密固定秘密消息：密钥由 128 字节随机 IV 与 128 字节私钥拼接，签名为 `IV || ciphertext`。验证端拒绝复用旧 IV，但只要构造一个不同 IV，使 RC4 KSA 在处理完前 128 字节后到达完全相同的状态，后半段私钥和最终密钥流就会保持不变，原密文也可继续使用。

## 解题过程

先请求一次签名并分离 `iv` 与 `ct`。RC4 KSA 第 $i$ 轮计算

$$
j_i=(j_{i-1}+S[i]+key[i])\bmod256
$$

并交换 $S[i]$、$S[j_i]$。前 128 轮只使用已知 IV，因此可以完整模拟并记录每轮的 $j_i$。

寻找 $i<j<k<128$，满足 $j_i=j$、$j_j=k$，且中间交换不会触碰 $i,j,k$。原过程在这三个位置上的两次关键交换为

$$
(i,j),(j,k),
$$

把它替换为

$$
(i,k),(j,i)
$$

后，两条路径都把初始排列 $(i,j,k)$ 变成 $(j,k,i)$。因此修改记录为 `new_j[i] = k`、`new_j[j] = i`，其余目标 $j$ 值保持不变。

给定目标 $j_i$ 序列，可以反向计算每个 IV 字节：

```python
def key_from_j_sequence(target):
    S = bytearray(range(256))
    previous_j = 0
    key = bytearray(len(target))
    for i, next_j in enumerate(target):
        key[i] = (next_j - previous_j - S[i]) % 256
        previous_j = next_j
        S[i], S[next_j] = S[next_j], S[i]
    return bytes(key)
```

新旧 IV 在第 128 轮结束时具有相同的 $S$ 排列和 $j$。随后 KSA 使用相同私钥，故最终 RC4 状态相同，PRGA 密钥流也相同。提交 `new_iv || old_ct`，既绕过 IV 黑名单，又让验证端解出原秘密消息，得到：

```text
grey{rc4_more_like_rcgone_amirite_q20v498n20}
```

## 方法总结

该方案把可控 IV 直接放在秘密密钥前，并把流密码加密误作签名。RC4 KSA 的交换序列存在可构造的状态碰撞，黑名单只比较 IV 字节值，无法阻止等价内部状态。认证协议应使用标准数字签名或 MAC；若使用带 nonce 的加密，也必须依赖经过证明的组合方式，而不能把“IV 不重复”当作真实性保证。
