# DownUnderCTF 2021 - That's Not My Name

## 题目简述

附件是一份疑似数据外传的 PCAPNG。流量中存在大量 DNS 查询，其中域名 `qawesrdtfgyhuj.xyz` 的子域标签异常长、字符只落在十六进制范围，并与普通 DNS 流量使用不同的服务器。目标是按抓包顺序恢复 DNS 查询名承载的外传字节流。

## 解题过程

在 Wireshark 中先筛选长 DNS 名称：

```text
dns && dns.qry.name.len > 35
```

可观察到大量指向 `qawesrdtfgyhuj.xyz` 的查询。为了只保留客户端查询、避免把响应中的同一名称再次拼入数据，用 `tshark` 提取：

```bash
tshark -r notmyname.pcapng \
  -Y 'dns.flags.response == 0 && dns.qry.name contains "qawesrdtfgyhuj.xyz"' \
  -T fields -e dns.qry.name > queries.txt
```

每行去掉固定域名后，剩余多个点分隔标签仍是十六进制片段。按文件原顺序删除点号并拼接、解码：

```python
suffix = ".qawesrdtfgyhuj.xyz"
hex_chunks = []

for line in open("queries.txt", encoding="utf-8"):
    name = line.strip().rstrip(".")
    if not name.endswith(suffix):
        continue
    encoded = name[:-len(suffix)].replace(".", "")
    hex_chunks.append(encoded)

payload = bytes.fromhex("".join(hex_chunks))
open("dns-exfil.bin", "wb").write(payload)
```

`binwalk dns-exfil.bin` 能识别出其中传输过的 PNG 等文件，说明重组方向正确；但不必先完整 carving，flag 本身以明文经过了 DNS 通道。直接搜索二进制中的可打印字符串：

```bash
grep -a -o 'DUCTF{[^}]*}' dns-exfil.bin
```

得到：

```text
DUCTF{c4t_g07_y0ur_n4m3}
```

## 方法总结

DNS 外传的常见信号包括高熵长子域、固定可疑后缀、异常查询频率以及只使用十六进制/Base32/Base64URL 字符。恢复时必须限定请求方向、保持包序并明确去除的固定域名，避免响应重复和字符串替换破坏数据。重组后先用文件识别和明文搜索做低成本验证，再决定是否需要进一步 carving。
