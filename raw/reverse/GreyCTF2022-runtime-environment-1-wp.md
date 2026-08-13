# GreyCTF2022 - Runtime Environment 1

## 题目简述

第一题给出带符号的静态 Go 编码器和 `challenge.txt`。程序行为类似 Base64，但使用自定义 64 字符表 `NaRvJT1B/.../M8itKe`，并以 `-` 代替标准填充字符。

## 解题过程

Go 符号未剥离，可直接定位 `main.main` 和 `main.Encode`。编码器仍按每 3 个输入字节拆成 4 个六位索引，因此只需把自定义字母表翻译回标准 Base64 表：

```python
import base64

custom = 'NaRvJT1B/m6AOXL9VDFIbUGkC+sSnzh5jxQ273d4lHPg0wcEpYqruWyfZoM8itKe-'
standard = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/='
table = str.maketrans(custom, standard)

data = open('challenge.txt').read().strip()
plain = base64.b64decode(data.translate(table))
print(plain.decode())
```

对仓库中的实际 `challenge.txt` 解码一次即得到：

```text
flag{B4s3d_G0Ph3r_r333333}
```

部分公开题解把外层前缀写成 `grey{...}`，但当前官方附件和 `flag.txt` 均为 `flag{...}`，此处以可复现的仓库数据为准。

## 方法总结

自定义 Base64 通常不改变 24 位分组算法，只替换输出字母表和填充符。通过不同长度输入观察“3 字节变 4 字符”的规律，再从二进制提取 64 字节常量，就能避免逐条阅读 Go 运行时代码。
