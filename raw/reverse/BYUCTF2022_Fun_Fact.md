# BYUCTF 2022 - Fun Fact

## 题目简述

附件 `obfuscated.py` 用 Base64 包住真正的 Python 程序并直接 `exec`。内层程序提供海洋生物随机知识，并把用户输入与一个单字节密钥异或后同固定密文比较。

## 解题过程

不要直接执行未知 Base64 载荷；先提取字符串并解码。内层关键代码是：

```python
random_array = xor("Snowflake eels have two sets of jaws",
                   "pretty crazy, huh?")
key = list(string.printable)[random_array[0] + random_array[8]]
encrypted = ''.join(chr(ord(x) ^ ord(key)) for x in user_input)
if encrypted == 'g%4c$zc%dz4gg;':
    print("Success!")
```

计算两个索引：

```text
'S' XOR 'p' = 35
'e' XOR 'r' = 23
35 + 23 = 58
string.printable[58] = 'W'
```

因此把目标密文逐字节与 `W` 异或：

```python
print(''.join(chr(ord(c) ^ ord('W')) for c in "g%4c$zc%dz4gg;"))
```

得到 flag 内文 `0rc4s-4r3-c00l`，最终提交：

```text
byuctf{0rc4s-4r3-c00l}
```

## 方法总结

外层 Base64 只是混淆。安全做法是静态解码并追踪密钥表达式；单字节 XOR 可直接逆运算，无需利用程序打印的加密 oracle，也无需运行包含递归输入流程的未知代码。
