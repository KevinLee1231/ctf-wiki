# AES Scissor Co. 3

## 题目简述

应用把用户对象序列化为 JSON，并以 AES-256-GCM 加密后放入 `session` Cookie。服务器仅在解密结果的 `role` 为 `admin` 时返回 flag。问题不在 AES 密钥，而在 12 字节 nonce 的生成方式：程序对“当前 Unix 秒”做 SHA-256，只取前 12 字节。同一秒内发起两次登录会复用相同 nonce，破坏 GCM 对 nonce 唯一性的要求。

Cookie 使用 URL-safe Base64，解码后的布局为 `nonce || ciphertext || tag`。页面响应头还会返回用户 UUID，使完整 JSON 明文可知。

## 解题过程

在同一秒内登录两次并断言 nonce 相同。两个明文形如：

```json
{"id":"<响应头中的 UUID>","role":"user","username":"test"}
```

GCM 的加密部分是计数器模式，因此同 nonce 下复用同一密钥流。选取等长目标明文，把角色改为 `admin`，并把用户名从 `test` 缩成 `tes` 维持总长度，即可得到伪造密文：

```python
target = f'{{"id":"{uuid1}","role":"admin","username":"tes"}}'.encode()
forged_ct = xor(ciphertext1, plaintext1, target)
```

仅修改密文还会被 GCM 标签拦截。对无 AAD 的情形，认证标签可写为

$$
T=\operatorname{GHASH}_H(C,L)\oplus S,
$$

其中 $H=E_K(0^{128})$，$S=E_K(J_0)$；二者在同 key、同 nonce 下都不变。把两组已知的 $(C_0,T_0)$、$(C_1,T_1)$ 相加（在 $GF(2^{128})$ 中加法就是 XOR）会消去 $S$：

$$
T_0\oplus T_1=\operatorname{GHASH}_H(C_0,L)\oplus\operatorname{GHASH}_H(C_1,L).
$$

将每个 16 字节块和长度块写成关于未知 $H$ 的多项式，在

$$
GF(2^{128})=GF(2)[x]/(x^{128}+x^7+x^2+x+1)
$$

上求根，便得到候选认证子密钥 $H$。注意最后一个不足 16 字节的密文块必须补零，长度块使用原始密文的比特长度。

对每个候选根，先由第一组样本恢复掩码

$$
S=T_0\oplus\operatorname{GHASH}_H(C_0,L),
$$

再计算伪造标签

$$
T'=\operatorname{GHASH}_H(C',L)\oplus S.
$$

最后按 `nonce || forged_ct || forged_tag` 重建 URL-safe Base64 Cookie。正确根会使服务器解密出管理员对象并返回：

```text
byuctf{n0m_n0m_c00k13_a4fb6c0f}
```

## 方法总结

- 核心技巧：利用 AES-GCM nonce 重用，同时恢复 CTR 密钥流关系和 GHASH 认证子密钥，从而伪造密文与标签。
- 识别信号：时间戳按秒生成 nonce、同秒可重复请求、完整明文可推导，都是 GCM 重用攻击的强信号。
- 复用要点：GCM 不是“有标签就绝对安全”；nonce 必须在同一密钥下唯一。实现攻击时要统一字段位序、零填充和长度块，否则有限域多项式即使思路正确也不会得到可用标签。
