# week2_re4

## 题目简述

题目是 PyInstaller 打包的 Python 逆向。`re4.exe` 中嵌入 Python 3.7 字节码，反编译后可见自定义 Base58 编码：输入字节串先按大端序转为大整数，再不断除以 58，用余数查自定义表输出。

目标是把硬编码 Base58 字符串按同一张表反向还原为字节串。

## 解题过程

### 关键观察

用 `pyinstxtractor-ng` 提取后得到 `re4.pyc`。`pycdc` 反编译可见：

```python
table = b"FG345EfghimnYZabcd67tuvwHJKLMNPopqrsUVWXjk12QRST89ABCDexyz"
enc = b"GkhwgiBYrd1aR2JJUmhkzEMKK6AQEcMzKcwpPiwKRCdsQA35bCJViDYz8iz"
```

编码逻辑是把输入 bytes 转大整数后 base58 编码，因此解码时按字符索引累乘 58。

### 求解步骤

```python
table = b"FG345EfghimnYZabcd67tuvwHJKLMNPopqrsUVWXjk12QRST89ABCDexyz"
enc = b"GkhwgiBYrd1aR2JJUmhkzEMKK6AQEcMzKcwpPiwKRCdsQA35bCJViDYz8iz"

num = 0
for ch in enc:
    num = num * 58 + table.index(ch)

flag = num.to_bytes((num.bit_length() + 7) // 8, "big").decode()
print(flag)
```

输出：

```text
flag{332f759f-9e38-4ddb-bdf1-eedf2a2658d0}
```

## 方法总结

- 核心技巧：PyInstaller 提取 `.pyc` 后反编译，识别大整数 Base58 编码。
- 识别信号：可执行文件体积较大、包含大量 `Py*` 符号或 PyInstaller 字符串时，应先按 Python 打包程序处理。
- 复用要点：Base58 变种的关键是字母表顺序；反解时按同一表索引累乘 58 即可。
