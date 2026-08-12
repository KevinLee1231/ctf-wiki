# DownUnderCTF 2022 legit-app-not-ransomware Writeup

## 题目简述

题目给出一个伪装成勒索软件的 .NET 6 单文件程序。它显示威胁信息、枚举用户目录下的 PNG 和 DOCX 文件，并要求输入解密口令。实际程序不会修改文件；flag 被拆成多个字段，经两层 Base64 关系隐藏在输入判断中。

## 解题过程

与普通托管程序不同，`LegitAppNotRansomware.exe` 的开头是 .NET 单文件宿主。文件偏移 `0x24600`（148992）处还有一个嵌入式 `MZ`，切出 8192 字节的托管程序集后即可反编译 `Program.cs`。

先确认程序行为：它等待 5 秒，然后递归查找 `C:\Users\<用户名>` 下的 `*.png` 和 `*.docx`，但循环中没有写入、重命名或删除操作。因此所谓“文件已加密”只是界面文案，不应真的执行恢复或支付流程。

校验条件可整理为：

```csharp
EncodeMe(Console.ReadLine()) == DecodeMe(d + u + c + t + f + "=")
```

其中 `EncodeMe` 是 Base64 编码，`DecodeMe` 是 Base64 解码。五段常量拼接并补上等号后为：

```text
UkZWRFZFWjdaREZrWDNrd2RWOXdZVzR4WTE4d2NsOWpNREJzWDJGelgyTjFZM1Z0WWpOeWZRPT0=
```

第一次解码得到：

```text
RFVDVEZ7ZDFkX3kwdV9wYW4xY18wcl9jMDBsX2FzX2N1Y3VtYjNyfQ==
```

因为左侧还对用户输入做了一次 Base64 编码，所以用户输入就是上面字符串再次解码后的内容：

```python
from base64 import b64decode

value = b'UkZWRFZFWjdaREZrWDNrd2RWOXdZVzR4WTE4d2NsOWpNREJzWDJGelgyTjFZM1Z0WWpOeWZRPT0='
print(b64decode(b64decode(value)).decode())
```

最终得到：

```text
DUCTF{d1d_y0u_pan1c_0r_c00l_as_cucumb3r}
```

## 方法总结

本题先用“勒索软件”外观制造压力，再用 .NET 单文件 bundle 和拆分字符串增加静态分析成本。应先通过反编译确认程序是否真的修改文件，再化简输入等式：右侧解码一次、左侧编码一次，意味着原始输入要对拼接常量解码两次。这样既能避免被威胁文案误导，也不会把枚举文件的无害代码误判成加密逻辑。
