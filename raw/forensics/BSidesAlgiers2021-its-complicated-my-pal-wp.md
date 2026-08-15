# It's Complicated My Pal

## 题目简述

题目给出一份约 38 MB、包含 52089 个数据包的抓包。协议统计中有 8378 个 ICMP 包，远多于正常网络中偶发的 ping；其中 `192.168.1.200` 发出的 Echo Request 数据区长度固定，内容却不断变化，符合用 ICMP payload 分片外带文件的特征。

目标是按抓包顺序连接这些 payload，恢复一个加密 ZIP，再通过字典攻击取得其中的图片。

## 解题过程

先解开外层归档并查看协议分布：

```bash
tar -xzf capture.tar.gz
capinfos capture.pcap
tshark -r capture.pcap -q -z io,phs
```

抽取指定源地址的 ICMP 数据并按包序直接连接：

```bash
tshark --disable-protocol hipercontracer \
  -r capture.pcap \
  -Y 'icmp && ip.src == 192.168.1.200' \
  -T fields -e data \
  | tr -d '\n' \
  | xxd -r -p > flag.zip

file flag.zip
```

输出应识别为 ZIP，开头为 `50 4b 03 04`，完整文件大小为 225571 字节。

这里的 `--disable-protocol hipercontracer` 是面向新版本 Wireshark 的兼容细节：部分随机 payload 恰好命中 HiPerConTracer dissector 后，通用 `data` 字段会变为空；若直接照旧版命令拼接，会少 31488 字节并得到损坏 ZIP。禁用该误识别协议后，全部 48 字节分片都能保留。在没有这个 dissector 的旧版本中，省略该选项也能得到相同结果。

ZIP 中只有一个 `flag.jpg`，但采用传统 ZipCrypto 加密。把哈希交给 John，并使用 `rockyou.txt`：

```bash
zip2john flag.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
john --show zip.hash
```

恢复出的密码为：

```text
craccer
```

解压并查看图片：

```bash
unzip -P craccer flag.zip
```

图片只是把最终结果印在一张梗图上，没有额外的像素、空间或隐写提取步骤，因此将关键信息直接转写为文本：

```text
shellmates{icmp_p@yl04d_4in't_us3l3ss_4ft3r_4ll_r1gHt?}
```

## 方法总结

本题的核心是网络会话之外的 ICMP 数据区重组。面对大量同方向、固定长度、内容高熵的 Echo Request，应先按源地址、方向和捕获顺序导出 payload，再用文件 magic 和容器结构验证结果；不能只在 Wireshark 中搜索 flag 字符串。

解析器也可能改变证据视图。现代 tshark 把随机数据误识别成上层协议时，字段为空不代表原始包没有数据，应对照帧长度和原始字节，必要时禁用相应 dissector。恢复出归档后，密码破解只是第二阶段，最终仍要验证解压文件类型和内容。
