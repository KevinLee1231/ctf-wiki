# week2EasyPcap

## 题目简述

题目要求从流量包中确定一次网络诊断操作及其目标 IP，再按照题面使用该操作名作为密钥，对目标 IP 做兼容 3DES 加密并以十六进制输出结果。

## 解题过程

在 Wireshark 中观察协议和 Info 列，可以看到成对出现的 ICMP Echo Request 与 Echo Reply，这正是 `ping` 的典型流量。题目询问主动探测的目标，因此应查看 Echo Request，而不是返回包；请求包的 Destination 为 `185.199.108.153`。

原 WP 使用的在线工具默认采用 ECB、Zero Padding，并将短密码按零字节补齐。密码 `ping` 不足一个 DES 密钥块，在该工具的兼容行为下结果等价于使用零补齐后的单 DES。下面的本地代码可以完整复现结果，不再依赖在线工具：

```python
from Crypto.Cipher import DES

plaintext = b"185.199.108.153"
plaintext = plaintext.ljust((len(plaintext) + 7) // 8 * 8, b"\x00")
key = b"ping".ljust(8, b"\x00")

ciphertext = DES.new(key, DES.MODE_ECB).encrypt(plaintext)
print(ciphertext.hex())
```

输出为：

```text
ec6d199865663767741e27953653206e
```

因此 flag 为：

```text
0xGame{ec6d199865663767741e27953653206e}
```

## 方法总结

流量侧的关键是区分 ICMP 请求与响应，并从请求包读取目标地址；加密侧必须明确模式、填充、密钥补齐和输出编码。只记录“使用某个在线 3DES 网站”无法复现结果，本题实际所需参数是 ECB、零填充、密钥 `ping`、明文 `185.199.108.153` 和 hex 输出。
