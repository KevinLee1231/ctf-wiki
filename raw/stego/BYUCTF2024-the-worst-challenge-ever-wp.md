# The Worst Challenge Ever

## 题目简述

附件表面是文本文件，开头和结尾分别对应 `hello, there` 与 `general kenobi`，中间却包含大量不可见的 `0x00` 和少量 `0x01`。隐藏信息由连续空字节的长度编码，因此按 Stego 归档。

## 解题过程

用十六进制查看可见，每个字符对应一段 `0x00`，再由一个 `0x01` 终止；该段零字节数量就是字符的 ASCII 码。逐字节计数即可：

```python
data = open("justterrible.txt", "rb").read()
start = data.index(b"hello, there") + len(b"hello, there")
end = data.index(b"general kenobi", start)

out = []
count = 0
for b in data[start:end]:
    if b == 0:
        count += 1
    elif b == 1:
        out.append(chr(count))
        count = 0

print("".join(out))
```

恢复结果为：

```text
byuctf{wh4ts_4_nu11_byt3_4nyw4ys}
```

## 方法总结

文本扩展名不代表内容一定是可打印字符。面对异常体积或大片 NUL，应切换到字节视角，观察分隔符与游程长度；本题的信道就是一元编码（unary coding），`0x01` 负责分帧，连续 `0x00` 的数量承载数值。
