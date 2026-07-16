# CBC

## 题目简述

题目实现了 8 字节分组的 CBC，但底层“分组密码”只是把每个字节与同一个一字节密钥异或，并非 AES。对第一个分组有 $C_1=P_1\oplus IV\oplus k$，密钥空间只有 256；既可以穷举，也可以利用已知 flag 前缀直接求出密钥。

## 解题过程

设上一块密文为 $C_{i-1}$，其中 $C_0=IV$。加密与解密关系为：

$$
C_i=(P_i\oplus C_{i-1})\oplus k,
\qquad
P_i=(C_i\oplus k)\oplus C_{i-1}.
$$

flag 首字节已知为字符 `0`，所以只看第一字节即可得到：

$$
k=C_1[0]\oplus IV[0]\oplus\operatorname{ord}('0').
$$

```python
iv = b"11111111"
enc = (
    b"\x8e\xc6\xf9\xdf\xd3\xdb\xc5\x8e8q\x10f>7.5"
    b"\x81\xcc\xae\x8d\x82\x8f\x92\xd9o'D6h8.d"
    b"\xd6\x9a\xfc\xdb\xd3\xd1\x97\x96Q\x1d{\\TV\x10\x11"
)

key = enc[0] ^ iv[0] ^ ord("0")

plain = bytearray()
prev = iv
for off in range(0, len(enc), 8):
    block = enc[off:off + 8]
    plain.extend(c ^ key ^ p for c, p in zip(block, prev))
    prev = block

pad_len = plain[-1]
if 1 <= pad_len <= 8 and plain[-pad_len:] == bytes([pad_len]) * pad_len:
    del plain[-pad_len:]

print(bytes(plain))
```

若不知道任何明文，也可令 `key` 遍历 `range(256)`，执行同样的 CBC 逆运算，再按可打印性或 flag 格式筛选结果。

## 方法总结

CBC 只规定分组之间如何链接，不会弥补底层算法过弱或密钥空间过小的问题。已知首块明文、IV 和首块密文时，可直接反推这种单字节异或密钥；实现解密时还应正确保留每个字节并验证、移除末尾填充。
