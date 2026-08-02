# TSGCTF2024 Feistel Barrier

## 题目简述

服务端使用 1024 位 RSA，并在明文前套了一层类似 OAEP 的双掩码结构：

```text
DB         = SHA256("") || PS || 0x01 || flag
maskedDB   = DB XOR MGF(seed, 95)
maskedSeed = seed XOR MGF(maskedDB, 32)
EM         = 0x00 || maskedSeed || maskedDB
c          = EM^e mod n
```

服务端公布 $n,e,c$，随后允许提交一个密文并直接返回 RSA 私钥运算得到的 128 字节 `EM`。它只拒绝与挑战密文完全相同的字节串，且即使命中拒绝提示，代码仍会继续解密。实际附件 `dist/server.py` 使用大端整数编码。

## 解题过程

RSA 的原始幂运算具有乘法同态。设挑战明文整数为 $m$、密文为 $c=m^e\bmod n$，构造：

$$c'=c\cdot2^e\bmod n$$

则解密得到：

$$m'=(c')^d\equiv2m\pmod n$$

提交代码为：

```python
chal_int = int.from_bytes(chal, 'big')
modified = chal_int * pow(2, e, n) % n
send(modified.to_bytes(128, 'big').hex())
```

`EM` 的首字节固定为零，所以 $m<2^{1016}$；题目生成的 1024 位模数远大于 $2m$，本实例不会发生模 $n$ 回绕。即使按通用写法处理，也可在模 $n$ 下乘 2 的逆元：

```python
m2 = int.from_bytes(bytes.fromhex(oracle_reply), 'big')
m = m2 * pow(2, -1, n) % n
EM = m.to_bytes(128, 'big')
```

得到完整编码后，Feistel 式掩码可以直接反向计算：

```python
masked_seed = EM[1:33]
masked_db = EM[33:]

seed_mask = mgf(masked_db, 32)
seed = xor(masked_seed, seed_mask)

db_mask = mgf(seed, 95)
DB = xor(masked_db, db_mask)
```

`MGF` 与服务端一致：对 `seed || counter.to_bytes(4, 'little')` 依次做 SHA-256 并截取所需长度。`DB` 的前 32 字节是空标签哈希，后面是若干零、分隔符 `0x01` 和 flag：

```python
flag = DB[32:].split(b"\x01", 1)[1]
```

恢复结果为：

```text
TSGCTF{B4|2ri3|2$$_4re_ju$$t_c|-|4ll3n63s_in_|)i$$6uise_5533d60b}
```

## 方法总结

题目表面加入了 OAEP 风格的 Feistel 掩码，但解密接口在校验填充前直接返回原始 RSA 明文块，使得 textbook RSA 的乘法同态完全暴露。只阻止原始挑战密文本身没有意义，攻击者可以构造相关密文并线性还原 $m$。安全实现必须使用成熟的 RSA-OAEP 解密 API，只在完整填充验证成功后返回统一结果，并避免把任何原始私钥运算输出作为 oracle 暴露。
