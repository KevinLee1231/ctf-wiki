# pypy

## 题目简述

附件给出了一段由 PyInstaller 打包的 Python 程序。其校验逻辑先去掉输入的 `hgame{}` 外壳，再交换相邻字符，最后让每个字节与自身下标异或，并把结果同一段十六进制密文比较。所有变换均可逆，可以按相反顺序恢复 flag。

## 解题过程

从打包程序中恢复 Python 字节码或源码后，可把加密过程概括为：

1. 去掉 flag 的固定前缀与右花括号；
2. 每两个字符交换一次位置；
3. 第 $i$ 个字节与整数 $i$ 异或；
4. 转成十六进制后与常量比较。

由于交换相邻字符执行两次会回到原序，逐字节异或同样是自逆运算，所以解密时先撤销下标异或，再交换相邻字节即可：

```python
cipher = bytearray.fromhex(
    "30466633346f59213b4139794520572b45514d61583151576638643a"
)

for index in range(len(cipher)):
    cipher[index] ^= index

for index in range(0, len(cipher), 2):
    cipher[index:index + 2] = cipher[index:index + 2][::-1]

print("hgame{" + cipher.decode() + "}")
```

输出为：

```text
hgame{G00dj0&_H3r3-I$Y@Ur_$L@G!~!~}
```

## 方法总结

PyInstaller 只改变程序的分发形式，不会让 Python 校验算法本身不可逆。恢复代码后，应按数据流逐层记录变换以及输入输出表示；对于相邻交换和异或这类自逆操作，解密只需逆序执行各层，避免在十六进制字符串与原始字节之间混淆。
