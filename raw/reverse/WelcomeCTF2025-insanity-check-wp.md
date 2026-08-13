# Insanity Check

## 题目简述

题目与 `Sanity Check` 共用服务。连接后表面上只显示欢迎横幅和第一枚 flag，但横幅之前还有 420 字节由空格、Tab 和换行组成的数据。它是一段 Whitespace 语言程序，执行后输出第二枚 flag。

## 解题过程

先无损保存服务原始字节，不能复制终端中“看起来为空”的区域：

```bash
nc HOST 33000 > textdump
```

从文件开头截取到第一个不属于 `0x20`、`0x09`、`0x0a` 的字节。Whitespace 只把这三种字符视为指令；本题程序主要重复使用“压栈整数”和“输出字符”，最后以三个换行结束。

也可以直接从官方服务源码中的 `white_spaces` 字符串做最小解释。下面的核心逻辑展示其指令形态：

```python
# Space Space <sign><bits> LF：压入整数
if code.startswith(b"  ", pos):
    pos += 2
    sign = code[pos]
    pos += 1
    bits = []
    while code[pos] != 0x0a:
        bits.append(0 if code[pos] == 0x20 else 1)
        pos += 1
    pos += 1
    value = int("".join(map(str, bits)) or "0", 2)
    stack.append(-value if sign == 0x09 else value)

# Tab LF Space Space：弹栈并按字符输出
elif code.startswith(b"\t\n  ", pos):
    pos += 4
    output.append(chr(stack.pop()))
```

完整执行程序得到：

```text
grey{7hEy_4Re_0r1Vin6_m3_1n54nE}
```

也可以把截取出的原始字节交给 [dCode Whitespace 解释器](https://www.dcode.fr/whitespace-language) 复核；正文已给出数据边界、语言识别和核心指令，不依赖外链才能理解解法。

## 方法总结

- 核心技巧：保存网络原始字节，识别只由 Space、Tab、LF 构成的 Whitespace 程序并执行。
- 识别信号：终端横幅前出现异常大空白，十六进制视图却有规律的 `20 09 0a` 序列，题名又暗示隐藏在签到题中的第二层内容。
- 复用要点：复制粘贴、自动去尾空格和文本格式化都会破坏程序；应从 PCAP、重定向文件或源码字符串中按字节提取。
