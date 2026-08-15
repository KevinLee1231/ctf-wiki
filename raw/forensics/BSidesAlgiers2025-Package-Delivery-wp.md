# BSidesAlgiers2025 - Package Delivery

## 题目简述

题目给出主机逻辑磁盘证据与网络抓包压缩包，要求重建一次 npm 依赖混淆攻击及其 NTP 隐蔽外传。完整链条包含 Web 日志、定时任务、恶意 npm 包、ZIP 密码、混淆的 `postinstall`、AES 密钥和 UDP/123 流量，不能从最后的 pcap 单独猜出全部上下文。

## 解题过程

### 从主机证据找到供应链入口

Web/应用日志中出现对 `/etc/cron.d/package-delivery` 与项目 `package.json` 的目录穿越读取。前者表明系统会周期性执行 `npm install`，后者显示依赖 `swiftpost-utilis` 使用宽松版本 `"*"`，而 npm registry 经 Verdaccio 代理到公共仓库。这形成依赖混淆条件：攻击者只要在公共 npm 发布同名高版本包，定时安装就会拉取并执行其 `postinstall`。

恶意包的安装命令把参数 `19d3d21c322b74c745f00aae61411d5a` 传给 `postinstall.js`。该值同时是证据包密码：

```bash
unzip -P 19d3d21c322b74c745f00aae61411d5a evidence.zip
```

解压后得到 `chall.pcap`。仓库中的 `decode.js` 可逐层撤销恶意脚本的 XOR 混淆，恢复 NTP 外传程序。对恢复二进制做静态检查，或根据其中的字符串定位开源 [ntpescape](https://github.com/evallen/ntpescape/) 实现，都能确认外传布局，并取得 AES-128 密钥：

```text
85c26059110c80a247adc4276bdf01e2
```

### 解码 NTP 外传

`solution/solve.py` 只处理目的端口为 UDP/123、长度至少 48 且 `payload[0] & 7 == 3` 的客户端包。NTP transmit timestamp 位于 `payload[40:48]`，其中低 17 位被用作隐蔽数据/长度信号；清掉这些位后与 8 个零字节拼成 AES-CTR 初始计数器：

```python
txsec, txfrac = struct.unpack(">II", payload[40:48])
counter = (
    struct.pack(">I", txsec)
    + struct.pack(">I", txfrac & ~0x1ffff)
    + b"\x00" * 8
)
cipher = AES.new(
    key,
    AES.MODE_CTR,
    nonce=b"",
    initial_value=int.from_bytes(counter, "big"),
)
plain = cipher.decrypt(payload[46:48])
```

每包实际携带 1 或 2 字节。脚本用 `txfrac` 的 bit 16 与同一 CTR 状态下零字节密文的最低位比较来恢复长度，然后按抓包顺序追加 `plain[:msglen]`。在解压出的 pcap 所在目录运行：

```bash
python3 solve.py
```

官方脚本与题解给出的最终结果为：

`shellmates{P4ckag3_del13vry_0V3r_NTP_i5_s0Oo_CoOo0l!!!}`

仓库还提供 `solve_replay.py`，可把 pcap 中的 UDP 载荷依次重放到本地接收器；它只是演示同一解码链，不是取得 flag 的必要条件。

## 方法总结

- 这是一条跨证据链：日志揭示定时 `npm install`，依赖配置解释恶意包为何被安装，`postinstall` 参数再解锁网络证据；只分析单一附件会丢失因果关系。
- NTP 时间戳既构造 AES-CTR 计数器又携带长度位，必须先清掉被占用的低位，再按大端序复现计数器。
- 公开实现的价值是确认协议细节，而不是用外链代替分析；正文已完整保留过滤条件、偏移、密钥、计数器构造和长度判定。
