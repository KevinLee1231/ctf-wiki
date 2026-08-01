# BYUCTF 2023 - kcpassword

## 题目简述

附件是 macOS 自动登录功能使用的 `kcpassword` 文件。该格式不是现代密码哈希，而是把明文与一个公开、固定的 11 字节密钥循环异或。

## 解题过程

固定密钥为：

```text
7d 89 52 23 d2 bc dd ea a3 b9 1f
```

逐字节循环异或即可还原；异或是自身的逆运算：

```python
key = bytes.fromhex('7d895223d2bcddeaa3b91f')
data = open('kcpassword', 'rb').read()
plain = bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
print(plain.rstrip(b'\x00').decode())
```

输出为：

```text
byuctf{wow_Macs_really_have_it_encrypted_with_a_static_key_lol}
```

## 方法总结

这是一项凭据工件恢复，而非爆破。识别专有文件名后应先查格式和已知常量；固定 XOR 密钥只能起到弱混淆作用，任何获得文件的人都能离线还原明文。
