# week2re4

## 题目简述

附件是 PyInstaller 打包的 Python 可执行文件。解包并反编译字节码后可以看到一段使用自定义 58 字符表的大整数编码；按该字符表执行 Base58 逆运算即可恢复 flag。

## 解题过程

可从 PyInstaller 归档标记、`PYZ`/`_MEIPASS` 等字符串识别打包方式。用 PyInstaller 解包器提取 PYZ 中的 `.pyc`，再选用与字节码版本匹配的反编译器恢复逻辑。关键点不是标准 Base58 字母表，而是程序内置的自定义 `table`。

逐字符计算 $s=s\times58+\operatorname{index}(c)$，最后按固定 42 字节大端序转回文本：

~~~python
encoded = 'GkhwgiBYrd1aR2JJUmhkzEMKK6AQEcMzKcwpPiwKRCdsQA35bCJViDYz8i'
table = 'FG345EfghimnYZabcd67tuvwHJKLMNPopqrsUVWXjk12QRST89ABCDexyz'
s = 0
for c in encoded:
    s *= 58
    s += table.find(c)
print(s.to_bytes(42, 'big').decode())
~~~

输出为：

```text
flag{332f759f-9e38-4ddb-bdf1-eedf2a2658d0}
```

## 方法总结

- 核心方法：识别并解包 PyInstaller，恢复 Python 字节码后按自定义字母表完成 Base58 解码。
- 识别特征：可执行文件含 PyInstaller 归档结构，反编译代码呈现“乘 58 再加字符下标”的循环。
- 注意事项：不能套用标准 Bitcoin Base58 字母表；固定输出长度会保留前导零字节，反编译器版本也必须匹配 `.pyc` 魔数。
