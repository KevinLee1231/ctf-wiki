# week2BabyRe

## 题目简述

附件包含 Python 2.7 字节码和编码结果。程序先对相邻 flag 字节做异或并加 30，再对整个结果进行 Base64 编码；已知首字节为字符 `0`，因此可按递推关系从前向后恢复明文。

## 解题过程

先根据 `.pyc` 魔数确认 Python 2.7 字节码版本，再使用匹配版本的反编译器恢复加密代码：

~~~python2
# uncompyle6 version 3.8.0
# Python bytecode 2.7 (62211)
# Decompiled from: Python 3.9.13 (main, Jun 8 2022, 09:45:57)
# [GCC 11.3.0]
# Embedded file name: 1.py
# Compiled at: 2022-08-22 17:01:47
import base64

def encode(str):
    fflag = '0'
    for i in range(1, len(flag)):
        x = ord(flag[i]) ^ ord(flag[i - 1])
        x += 30
        fflag += chr(x)
    return base64.b64encode(fflag)

flag = open('flag.txt').read()
enc = encode(flag)
print enc
# okay decompiling flag.pyc
~~~

Base64 解码后的字节记为 $y_i$，原 flag 字节记为 $f_i$。程序给出 $f_0=\texttt{'0'}$，并满足：

$$
y_i=(f_i\oplus f_{i-1})+30,\quad i\ge1
$$

因此：

$$
f_i=f_{i-1}\oplus(y_i-30)
$$

按该递推解码：

~~~python
import base64

cipher = open("out.txt", "rb").read().strip()
raw = base64.b64decode(cipher)
flag = bytearray(b"0")
for i in range(1, len(raw)):
    flag.append(flag[i - 1] ^ (raw[i] - 30))
print(flag.decode())
~~~

得到 flag：

~~~text
0xGame{afd9461d-35fb-4e9b-9716-aa83b3ed681a}
~~~

## 方法总结

- 核心方法：反编译 PYC 恢复相邻字节递推，用已知首字符逐位逆运算，再去除外层 Base64。
- 识别特征：编码循环使用当前字节与前一字节异或，输出的第一个字节被固定为已知值。
- 注意事项：先确认 Python 字节码版本；Python 3 中遍历 `bytes` 已直接得到整数，不应再对其调用 `ord()`，并应使用相对附件路径而非原作者桌面路径。
