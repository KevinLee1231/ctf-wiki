# Easy Money

## 题目简述

二进制中放入了大量很长的随机字符串作为干扰，但程序实际引用的秘密位于最后一个全局字符串。该字符串只经过 Base64 编码，没有参与复杂运算。

## 解题过程

用 `strings` 或反编译器查看只读数据，可以看到多个 `string1` 至 `string10`。前九项主要是歌词和随机字符，`string10` 为：

```text
VU1EQ1RGLXtlNHN5X3cxbnNfZTRzeV9tMG4zeX0=
```

直接解码：

```python
import base64

encoded = b"VU1EQ1RGLXtlNHN5X3cxbnNfZTRzeV9tMG4zeX0="
print(base64.b64decode(encoded).decode())
```

得到：

```text
UMDCTF-{e4sy_w1ns_e4sy_m0n3y}
```

## 方法总结

静态数据很多时，应结合交叉引用判断哪些字符串真正进入控制流。即使不做完整反编译，标准 Base64 的字符集、长度和 `=` 填充也足以把最后一项从随机干扰中筛出。
