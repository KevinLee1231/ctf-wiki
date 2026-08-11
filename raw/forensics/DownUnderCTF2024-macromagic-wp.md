# Macro Magic

## 题目简述

附件包括启用宏的工作簿 `Monke.xlsm` 和对应的 HTTP 流量。宏从未提供的 `flag.xlsx` 读取 `A1`，加上 `Flag: ` 前缀后用循环 XOR 加密，再把密文字节转十进制并把空格替换为连字符，通过 HTTP GET 外传。题目核心是从宏语义和 PCAP 重建编码链，归入 Forensics。

## 解题过程

从工作簿中提取 VBA 后，主过程的关键数据流是：

1. `valueA1 = wb.Sheets(1).Range("A1").Value` 读取 flag；
2. `Q = "Flag: " & valueA1`；
3. `W = S + G + D + F`，四段拼成 XOR 密钥 `MonkeyMagic`；
4. `doThing(Q, W)` 对每个字符与循环密钥 XOR；
5. `forensics` 将 XOR 结果逐字节转换为十进制数、以空格分隔；
6. `totalyFine` 把空格替换为 `-`，`superThing` 将结果拼到 URL 后发出 GET。

从 PCAP 导出的 HTTP 对象中找到有意义的连字符十进制串：

```text
11-3-15-12-95-89-9-52-36-61-37-54-34-90-15-86-38-26-80-19-1-60-12-38-49-9-28-38-0-81-9-2-80-52-28-19
```

逆向过程正好反过来：把 `-` 换为空格、把十进制数恢复为字节，再与循环密钥 `MonkeyMagic` XOR。等价伪代码如下：

```python
values = "...".replace("-", " ").split()
encrypted = bytes(int(value) for value in values)
key = b"MonkeyMagic"
plain = bytes(byte ^ key[i % len(key)] for i, byte in enumerate(encrypted))
print(plain.decode())
```

结果为 `Flag: DUCTF{M4d3_W1th_AI_by_M0nk3ys}`，因此提交：

```text
DUCTF{M4d3_W1th_AI_by_M0nk3ys}
```

## 方法总结

带宏 Office 文档的网络证据不能孤立看：宏说明“字段如何变换”，PCAP 提供“实际变换后的数据”。先画出变量从明文到 URL 的数据流，再把捕获值倒放回去，能排除大量故意制造的假 flag、梗图和无关 HTTP 请求。
