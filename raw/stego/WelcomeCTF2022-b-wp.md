# B

## 题目简述

附件是一张可以正常播放的 GIF。真正载荷位于 GIF 结束标记 `0x3B` 之后，由字符 `»` 与 `B` 表示二进制位。关键不是动图画面，而是识别并解码文件尾的附加数据。

## 解题过程

十六进制查看文件尾，可以看到正常 GIF 数据之后出现长串重复字符。按照官方题解，将 `»` 映射为 `0`、`B` 映射为 `1`，每 8 位还原一个字节：

```python
from pathlib import Path

data = Path("nic_cage.gif").read_bytes()
trailer = data.rfind(b"\x3b")
tail = data[trailer + 1:].decode("latin-1")

bits = tail.replace("»", "0").replace("B", "1")
assert len(bits) % 8 == 0
plain = bytes(int(bits[i:i + 8], 2) for i in range(0, len(bits), 8))
print(plain.decode())
```

这里使用 `latin-1` 是因为附件中的 `»` 以单字节 `0xBB` 保存，不是 UTF-8 的双字节编码。输出为：

```text
greyhats{0h_nO_No7_7H3_Bs!!!}
```

## 方法总结

媒体文件尾随数据常被常规查看器忽略。先依据格式结束标记划分“有效媒体”和“附加载荷”，再观察尾部字符集是否能映射到二进制。解码前要确认字符编码和位数是否为 8 的倍数，否则简单的文本解码就可能改变原始字节并产生错位。
