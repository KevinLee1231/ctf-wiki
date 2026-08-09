# playing_with_xor

## 题目简述

附件给出一段由短密钥重复异或得到的密文。flag 格式固定为 `n00bz{...}`，因此开头至少有 7 个已知明文字节，恰好足以恢复 7 字节周期密钥。

## 解题过程

重复密钥异或满足 $c_i=p_i\oplus k_{i\bmod 7}$。取密文前 7 字节与已知前缀 `n00bz{p` 异或，即可得到完整周期密钥；再循环使用该密钥解密全部字节：

```python
known = b"n00bz{p"
key = bytes(c ^ p for c, p in zip(ciphertext[:7], known))
plain = bytes(c ^ key[i % len(key)] for i, c in enumerate(ciphertext))
print(plain.decode())
```

输出为：

```text
n00bz{pl4y1ng_w1th_x0r_m0r3_l1k3_pl4y1ng_w1th_f1r3}
```

## 方法总结

重复密钥异或的安全性取决于密钥周期不会被已知明文覆盖。本题的 flag 格式直接提供了一个完整周期，因而一次异或就泄露全部密钥。处理这类题时应先测试周期长度，再利用格式、文件头或自然语言高频片段构造已知明文。
