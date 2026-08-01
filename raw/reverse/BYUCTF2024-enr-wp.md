# enr

## 题目简述

题目给出 Vilo 路由器 Android APK 与配网流量，要求恢复客户端发送的非空 `enr` 字段。协议消息经过自定义 4 字节分组倒序、APK 内的 XXTEA/Block TEA 变体以及第二层 XXTEA 加密；决定性工作是从应用逻辑还原完整密钥派生与解密顺序。

## 解题过程

在 PCAP 的同一 TCP 流中取第二条服务端消息和后续客户端数据。协议头长度并不相同：密钥消息跳过前 16 字节，数据消息跳过前 15 字节。APK 的 deobfuscation 会分别反转每个 4 字节块：

```python
def deobfuscate(buf):
    return b"".join(buf[i:i + 4][::-1] for i in range(0, 16, 4))
```

第一阶段固定 key 在反转前表现为 `routerLocalWhoAr`，Java 代码实际使用 `tuoroLreWlacrAoh`。对密钥消息执行分组倒序后，输入 `pain.java` 的参数为：

```text
e57237829115616cc9083552610aaab5
```

自定义 XXTEA 解密输出：

```text
E13A4ADBBF2E8161F5EF1111CDE7D5FD
```

再做一次 4 字节分组倒序，得到第二阶段 128 位 key：

```text
db4a3ae161812ebf1111eff5fdd5e7cd
```

最后对数据消息去掉 15 字节头，以该 key 做 XXTEA 解密。恢复的 JSON 中非空 `enr` 值为：

```text
IQY6coUvUBQ8NOHw
```

最终提交 `byuctf{IQY6coUvUBQ8NOHw}`。

## 方法总结

移动端私有协议题要把 PCAP 分帧、APK 常量、字节序变换和密码层次逐项对齐。每一阶段都保留中间十六进制值，可以准确区分“抓包偏移错”“分组反转错”和“XXTEA key 错”，避免只用最终 flag 反推过程。
