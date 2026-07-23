# Roy's Randomness

## 题目简述

题目给出一份 TCP 抓包。发送端用不同 TCP 标志位表示摩尔斯符号，再把摩尔斯结果解释成十六进制，最后转回 ASCII，构成多层协议隐蔽信道。

## 解题过程

按数据包时间顺序读取三个标志：

```text
SYN -> .
RST -> -
PSH -> 分隔符
```

每遇到 PSH，就结束当前摩尔斯码组。将点划组合查摩尔斯表，可得到一个十六进制字符；每两个十六进制字符组成一个字节，再按 ASCII 解码。

伪代码如下：

```python
symbols = []
current = ""
for packet in packets:
    if packet.tcp.syn:
        current += "."
    elif packet.tcp.rst:
        current += "-"
    elif packet.tcp.psh:
        symbols.append(morse_to_char[current])
        current = ""

plain = bytes.fromhex("".join(symbols)).decode()
```

输出为：

```text
UMDCTF-{r0y_f0und_m0r53}
```

## 方法总结

本题的编码链是“TCP 标志位 → 摩尔斯码 → 十六进制 → ASCII”。分析时应逐层保存中间结果，先确认点划分隔，再验证十六进制字符范围，最后才转字节。若同时读取双向流或重传包，会破坏符号顺序，应限定正确会话方向并按帧序去重。
