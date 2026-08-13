# GreyCTF2023 EncryptService

## 题目简述

服务使用固定 AES 密钥和 CTR 模式加密数据，nonce 定义为 `SHA256(iv)[:8]`。用户能指定明文，但 `iv` 只有一个字节；服务还会用随机单字节 `iv` 加密 flag。由于单字节空间只有 256 种，所有可能的密钥流都能被枚举。

## 解题过程

CTR 加密满足 $C=P\oplus S(K,nonce)$。向服务提交一段与 flag 等长的全零明文，并让服务依次使用 `00` 到 `ff` 的全部 `iv`，得到的密文就是 256 条候选密钥流：

```python
known = b"\x00" * flag_len
streams = []
for iv in range(256):
    streams.append(encrypt(known, bytes([iv])))
```

对 flag 密文逐一异或候选流，筛选可打印文本且以 `grey{` 开头的结果：

```python
for stream in streams:
    plain = bytes(a ^ b for a, b in zip(flag_ct, stream))
    if plain.startswith(b"grey{"):
        print(plain.decode())
```

恢复出：

```text
grey{0h_m4n_57r34m_c1ph3r_n07_50_53cur3}
```

## 方法总结

CTR 的安全性依赖同一密钥下的 nonce 不重复且不可被穷举。对 nonce 再做哈希并不会扩大原始熵：输入只有 8 位，输出仍然只有 256 个可能值。已知明文查询可直接提取密钥流，因此本题本质是“小 nonce 空间 + 固定密钥”的完全枚举。
