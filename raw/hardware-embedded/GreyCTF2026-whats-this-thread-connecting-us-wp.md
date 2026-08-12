# What's this Thread Connecting Us?

## 题目简述

题目使用三块 GreyFlag ESP32-C6 徽章：选手徽章加入现场 OpenThread 网络，另一块刷成 802.15.4 嗅探器，组织者设备周期广播加密 flag。需要取得 Thread NetworkKey，筛出目的 PAN ID 为 `0x1337` 的安全帧，并按 Thread 密钥派生和 IEEE 802.15.4 CCM* 格式解密。

## 解题过程

先在徽章菜单进入挑战 4 的 OpenThread CLI，启动接口并以公开 joiner 凭据加入网络：

```text
ot ifconfig up
ot joiner start GREYFLAG99
ot thread start
ot dataset active
```

`ot dataset active` 会打印当前 Active Operational Dataset，其中的 `Network Key` 是 16 字节十六进制值，后续必须原样保留。

将提供的 `sniffer_merged.bin` 刷入另一块徽章并连接串口：

```bash
esptool.py -p <serial-port> write_flash 0x0 sniffer_merged.bin
```

嗅探输出每行包含一个原始 802.15.4 帧。目的 PAN ID 位于线偏移 3、4，采用小端表示，因此筛选字节 `37 13`。目标帧的安全控制字段为 `0x15`：安全级别 5（ENC-MIC-32），Key ID mode 2。

解析时先去除 ESP-IDF 接收端附加的 2 字节 RSSI/LQI 元数据，再根据 Frame Control 中的地址模式和 PAN compression 位前移游标。目标帧的关键布局为：

| 偏移 | 字段 |
| ---: | --- |
| 0–1 | Frame Control，小端 |
| 2 | Sequence Number |
| 3–4 | Destination PAN ID `0x1337` |
| 5–6 | 广播短地址 `0xffff` |
| 7–14 | Source EUI-64 |
| 15 | Security Control |
| 16–19 | Frame Counter，小端 |
| 20–23 | Key Source |
| 24 | Key Index |
| 25–末尾 | ciphertext 与 4 字节 MIC |

Thread 不直接把 NetworkKey 作为 AES key。MAC key 的派生方式是：

$$
K_{material}=\operatorname{HMAC\text{-}SHA256}
(K_{network},\ KeySource\parallel\texttt{"Thread"})
$$

$$
K_{MAC}=K_{material}[16:32]
$$

```python
material = hmac.new(
    network_key,
    key_source + b"Thread",
    hashlib.sha256,
).digest()
mac_key = material[16:32]
```

CCM 的 13 字节 nonce 为：

$$
nonce=SrcEUI64\parallel FrameCounter_{LE}\parallel SecurityLevel
$$

从 Frame Control 到 Key Index 的前 25 字节全部是 AAD；其后除末 4 字节外是密文，末 4 字节是 MIC-32：

```python
nonce = src_eui64 + frame_counter_raw + bytes([5])
aad = frame[:25]
ciphertext = frame[25:-4]
mic = frame[-4:]

cipher = AES.new(mac_key, AES.MODE_CCM, nonce=nonce, mac_len=4)
cipher.update(aad)
plaintext = cipher.decrypt_and_verify(ciphertext, mic)
```

仓库说明记录了一个现场固件问题：广播端附加 FCS 时可能覆盖 MIC 最后两字节，嗅探端又把这两字节替换为 RSSI/LQI。遇到这种已知损坏的抓包时，`decrypt_and_verify` 会失败，但 CCM 密钥流本身不依赖接收到的 tag，可以在明确标注“未完成认证”的前提下只调用 `decrypt(ciphertext)` 恢复内容；若 MIC 完整，则必须优先使用验证路径。

正确帧解密为：

```text
grey{B0und_By_Akai_Ito}
```

## 方法总结

完整链路包含三种不同密钥/字段语义：OpenThread dataset 提供 NetworkKey，HMAC 派生出链路层 MAC key，802.15.4 帧头再提供 CCM nonce 与 AAD。最容易出错的是端序、KeySource 后是否附加 NUL、MAC key 取 HMAC 的哪一半，以及把接收元数据误当成 MIC。认证失败时可以因题目已知 FCS 缺陷做诊断性解密，但必须明确这只证明明文可恢复，不证明帧完整性。
