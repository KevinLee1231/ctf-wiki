# DownUnderCTF 2022 oracle for block cipher enthusiasts Writeup

## 题目简述

服务在同一连接中生成一个 AES 密钥，并允许用户为同一条消息选择两次 OFB 模式的 IV。消息首块固定为 `Decrypt this... `，后面是随机十六进制字符串和 flag。

OFB 的密钥流为：

$$
K_1=E_k(IV),\qquad K_{i+1}=E_k(K_i),\qquad C_i=P_i\oplus K_i.
$$

用户能控制第二次查询的 IV，因此可以把第一次查询泄露出的中间状态重新送入分组密码。

## 解题过程

第一次选择任意 16 字节 `IV`。由已知首块明文和首块密文得到：

$$
E_k(IV)=P_0\oplus C^{(1)}_0.
$$

第二次直接把这个值作为 IV。其首块密钥流就是 $E_k^2(IV)$，再次利用固定首块即可恢复：

$$
E_k^2(IV)=P_0\oplus C^{(2)}_0.
$$

现在可以解开第一次密文的第二块。恢复出的该块明文又与第二次密文的同位置块异或，得到下一轮密钥流；如此迭代即可恢复剩余消息：

```python
known = b"Decrypt this... "
iv = b"A" * 16

ct1 = query(iv)
e_iv = xor(known, ct1[:16])

ct2 = query(e_iv)
keystream = xor(known, ct2[:16])  # E_k^2(iv)

plaintext = bytearray()
for i in range(1, (len(ct1) + 15) // 16):
    block = ct1[16*i:16*(i+1)]
    p = xor(block, keystream[:len(block)])
    plaintext += p
    keystream = xor(p, ct2[16*i:16*(i+1)])
```

去掉随机十六进制字段后得到：

```text
DUCTF{0fb_mu5t_4ctu4lly_st4nd_f0r_0bvi0usly_f4ul7y_bl0ck_c1ph3r_m0d3_0f_0p3ra710n_7b9cb403e8332c980456b17a00abd51049cb8207581c274fcb233f3a43df4a}
```

## 方法总结

OFB 本身并未失效，问题在于服务让攻击者自选 IV，并在同一密钥下加密相同且有已知前缀的消息。已知明文泄露 $E_k(IV)$，而自选下一 IV 又把这个内部状态变成新的输入，于是可以逐块追踪整个密钥流。设计加密 oracle 时，IV 不仅要唯一，还要防止攻击者把内部状态作为下一次查询输入并获得相关消息的已知明文反馈。
