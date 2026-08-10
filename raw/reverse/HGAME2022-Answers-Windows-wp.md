# Answer's Windows

## 题目简述

题目是一个 Qt 图形程序，输入正确时加载 `right.png`，错误时加载 `wrong.png`。校验逻辑包含少量反调试代码和一套修改过字母表的 Base64 编码；程序要求编码结果长度为 56，并与内置字符串比较。

## 解题过程

先在二进制中搜索 `right.png` 或 `wrong.png`，沿交叉引用定位到决定界面结果的函数。成功分支前有如下判断：

```c
if (size == 56 &&
    !memcmp(encoded,
            ";'>B<76\\=82@-8.@=T\"@-7ZU:8*F=X2J<G>@=W^@-8.@9D2T:49U@1aa",
            0x38)) {
    // 加载 right.png
}
```

继续跟进编码函数，可以确认主体仍是 Base64，只是索引表被替换为从 ASCII `!` 开始连续排列的字符：

```text
!"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`a
```

这里共有 65 个字符。前 64 个字符对应标准 Base64 的 64 项字母表，最后的 `a` 对应填充符 `=`。因此密文结尾的 `aa` 必须还原为 `==`；如果把它们当成普通字母解码，flag 后会多出两个垃圾字节。

直接建立翻译表并解码：

```python
import base64

cipher = ";'>B<76\\=82@-8.@=T\"@-7ZU:8*F=X2J<G>@=W^@-8.@9D2T:49U@1aa"

custom_alphabet = "".join(chr(code) for code in range(33, 98))
standard_alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/="

translated = cipher.translate(
    str.maketrans(custom_alphabet, standard_alphabet)
)
print(base64.b64decode(translated).decode())
```

输出为：

```text
hgame{qt_1s_s0_1nteresting_so_1s_b4se64}
```

## 方法总结

Qt 或 C++ 标准库会让反编译结果显得冗长，但本题的有效路径很短：由成功、失败界面资源定位比较点，再从编码函数对字母表的引用确认变种 Base64。恢复自定义编码时要连同填充规则一起检查；仅替换 64 个数据字符而忽略第 65 个填充字符，会得到尾部含垃圾字节的近似答案。
