# 码海舵师

## 题目简述

程序读取用户输入，经自实现的标准 Base64 编码函数处理后，与内置字符串比较。因为 Base64 是可逆编码，直接解码目标字符串就能得到应输入的 flag。

## 解题过程

IDA 中 `sub_401310` 接收用户输入并产生编码结果，调用者随后把结果与以下常量比较：

```text
MHhHYW1le2ZjYmVlZWM3LTc3NTgtYmEyZi1jMDU5LTJmNWNhOWEzODc5YX0=
```

字符串只使用标准 Base64 字符表并以 `=` 填充，可直接用标准库解码：

```python
from base64 import b64decode

target = b"MHhHYW1le2ZjYmVlZWM3LTc3NTgtYmEyZi1jMDU5LTJmNWNhOWEzODc5YX0="
print(b64decode(target).decode())
```

输出为：

```text
0xGame{fcbeeec7-7758-ba2f-c059-2f5ca9a3879a}
```

也可以把解码结果重新执行 Base64 编码，并确认输出与程序内置常量完全相同，从而交叉验证转写无误。

## 方法总结

识别编码不能只看末尾 `=`，还要结合字符表、数据长度和程序中的编码循环确认。目标是“编码后等于常量”时，对常量执行逆编码即可；正文已给出完整常量和标准库代码，不依赖在线工具。
