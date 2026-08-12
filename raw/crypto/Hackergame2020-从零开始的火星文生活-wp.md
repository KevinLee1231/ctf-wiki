# Hackergame2020 从零开始的火星文生活 WP

## 题目简述

附件是 UTF-8 文本，但可见内容像“脦脪鹿楼”一类乱码。它不是一次简单的“选错编码”，而是 GBK、ISO-8859-1 与 UTF-8 之间经过多次错误解码、重新编码后形成的可逆链；恢复后还要把全角 flag 转回 ASCII。

## 解题过程

官方生成器先把 ASCII 可打印字符变成全角字符，再执行：

```python
output = (
    fullwidth_text
    .encode("gbk")
    .decode("iso-8859-1")
    .encode("utf-8")
    .decode("gbk")
)
```

最终 `output` 又以 UTF-8 写入附件。逆向时按相反顺序执行：

```python
def to_ascii(text):
    out = []
    for char in text:
        code = ord(char)
        if code == 0x3000:
            out.append(" ")
        elif 0xFF01 <= code <= 0xFF5E:
            out.append(chr(code - 0xFEE0))
        else:
            out.append(char)
    return "".join(out)

with open("gibberish_message.txt", encoding="utf-8") as file:
    damaged = file.read()

decoded = damaged.encode("gbk").decode("utf-8")
decoded = decoded.encode("iso-8859-1").decode("gbk")
print(to_ascii(decoded))
```

第一步把“以 GBK 字符解释过的 UTF-8 字节”还原，第二步把“以 ISO-8859-1 字符解释过的 GBK 字节”还原。全角字符与 ASCII 在常用可打印范围内相差 `0xFEE0`。最终得到：

```text
flag{H4v3_FuN_w1Th_3nc0d1ng_4Nd_d3c0D1nG_9qD2R8hs}
```

官方“常见乱码”表只是把乱码类型画成表格，正文已经完整写出每层编码方向，因此不再保留截图。

## 方法总结

乱码恢复应把每一步区分为“字符到字节的 encode”和“字节到字符的 decode”，并从最终可见文本反向撤销。不要只在编辑器里盲试编码；把转换链写成代码后，每一层的字节是否可解码、是否无损都可以被明确验证。
