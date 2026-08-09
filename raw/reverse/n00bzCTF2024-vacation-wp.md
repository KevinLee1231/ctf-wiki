# Vacation

## 题目简述

PowerShell 脚本逐字节把 flag 与常数 3 异或，附件同时给出输出。异或具有自反性，对密文字节再次异或 3 即可恢复原文。

## 解题过程

脚本核心是：

```powershell
$newBytes.Add($_ -bxor 3)
```

用任意按字节处理的实现逆变换：

```python
data = open("output.txt", "rb").read().rstrip(b"\r\n")
print(bytes(value ^ 3 for value in data).decode("ascii"))
```

输出：

```text
n00bz{from_paris_wth_xor}
```

## 方法总结

固定单字节 XOR 不需要求密钥，题目已在脚本中明示常数。读取时应保留原始字节，并只去掉 `Out-File` 追加的行尾，避免文本转码改变密文。
