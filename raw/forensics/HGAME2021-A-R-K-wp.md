# A R K

## 题目简述

附件是一份同时包含 FTP 与 TLS 会话的 PCAP。FTP 明文传输了 TLS 会话密钥日志 `ssl.log`，可用它解密前面的 HTTPS 流量；HTTP 响应中的明日方舟战斗回放又经过 JSON、Base64 和损坏 ZIP 头三层包装。修复并解包后，需要把部署记录中的坐标绘成二维码。整条链的核心是从网络证据中逐层恢复文件与操作记录，因此归入 Forensics。

## 解题过程

用 Wireshark 打开抓包，后半段可见 FTP 控制流和 `FTP-DATA`。找到上传或下载 `ssl.log` 的数据连接，选择“追踪 TCP 流”，将原始内容另存为 `ssl.log`。该文件不是普通日志，而是浏览器或客户端导出的 TLS key log，包含 `CLIENT_HANDSHAKE_TRAFFIC_SECRET`、`SERVER_TRAFFIC_SECRET_0` 等会话秘密。

在 Wireshark 的 TLS 协议设置中，把 `(Pre)-Master-Secret log filename` 指向该文件，重新解析抓包后，原来的 TLS Application Data 会还原为 HTTP。再使用“导出对象 → HTTP”查看 JSON 响应。`getBattleReplay` 出现在 `battleStart` 之前，符合游戏“自律回放”的语义；同名对象中应选择服务端返回给客户端、`Content-Type` 为 JSON 的那一份。

导出的 JSON 含有 `battleReplay` 字段，其值为 Base64。先解码为二进制：

```python
from base64 import b64decode
from pathlib import Path

import json5


source = Path("getBattleReplay.json")
data = json5.loads(source.read_text(encoding="utf-8"))
Path("replay.bin").write_bytes(b64decode(data["battleReplay"]))
```

`replay.bin` 以 `50 4B` 开头，说明它本应是 ZIP，但前四字节被改坏。把文件开头修复为标准本地文件头 `50 4B 03 04` 后即可解压，得到 `default_entry`；其内容仍是 JSON5。关键数组是 `journal.logs`，每条记录的 `pos.row` 与 `pos.col` 是一次干员部署坐标。

题解中的提示“星藏点雪，月隐晦明，以夕为墨，泼雪作屏”说明以白色为底、把记录坐标画成黑点。完整绘图代码如下：

```python
from pathlib import Path

import json5
import numpy as np
from PIL import Image


source = Path("default_entry.json")
replay = json5.loads(source.read_text(encoding="utf-8"))

canvas = np.full((100, 100, 3), 255, dtype=np.uint8)
for record in replay["journal"]["logs"]:
    row = record["pos"]["row"]
    column = record["pos"]["col"]
    canvas[row, column] = (0, 0, 0)

Image.fromarray(canvas, mode="RGB").save("dusk-qr.png")
```

绘制结果是二维码：

![根据战斗回放中的部署坐标绘制出的黑白二维码](./HGAME2021-A-R-K-wp/dusk-qr.png)

扫描后得到：

```text
hgame{Did_y0u_ge7_Dusk?}
```

## 方法总结

本题的每一层都给下一层提供解码材料：FTP 流恢复 TLS key log，TLS key log 解密 HTTP，HTTP 对象给出 Base64 回放，文件魔数指向 ZIP 头修复，回放 JSON 的坐标再转成二维码。处理流量题时应优先寻找明文协议传输的密钥、配置或附件；提取二进制后再依靠魔数和数据语义继续，而不是停在某一种工具界面。PDF 中的 Wireshark、十六进制编辑器和代码截图均已转写为文字与代码，只保留最终二维码这一项具有视觉验证价值的结果。
