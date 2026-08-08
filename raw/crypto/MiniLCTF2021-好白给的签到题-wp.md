# MiniLCTF 2021 - 好白给的签到题

## 题目简述

附件解压后得到一段多层嵌套文本。每层都是 Base64，但部分层在编码后又把整个字节串反转。题目给出的层数按斐波那契式增长，手工逐层尝试很容易出错，因此需要用“正向解码失败就反转后解码”的循环自动剥离。

## 解题过程

Base64 字符集固定，正常层可以直接解码；反转层通常会因为填充符 `=` 跑到开头或解码结果不是 UTF-8 而暴露。每轮同时尝试原串与逆序串，选择能得到可读 UTF-8 的结果，直到出现 flag 格式：

```python
from base64 import b64decode

data = open("story.txt", "rb").read().strip()

while True:
    candidates = []
    for encoded in (data, data[::-1]):
        try:
            plain = b64decode(encoded, validate=True)
            plain.decode("utf-8")
            candidates.append(plain)
        except Exception:
            pass

    if not candidates:
        raise RuntimeError("当前层既不是正向 Base64，也不是反向 Base64")

    data = candidates[0].strip()
    text = data.decode("utf-8")
    if "{" in text and "}" in text:
        print(text)
        break
```

最终得到：

```text
MiniLCTF{5o_m@ny_Inn3r5p4ce_hs!!}
```

## 方法总结

多层编码题不应靠网页工具反复复制。先从字符集、填充位置和解码后可打印性判断当前方向，再循环到明确的终止条件。这里的核心障碍是 Base64 与字节串反转这两种可逆表示变换，因此归入密码学方向；脚本只负责求解单题，不用于批量改写 WP。
