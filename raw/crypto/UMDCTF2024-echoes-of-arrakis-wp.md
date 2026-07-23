# Echoes of Arrakis

## 题目简述

题目给出一段 WebM 影片和配套 SRT 字幕，并提示 flag 使用小写字母和下划线。视频正文只是介绍 Fremen；真正异常出现在字幕中：`(ARE KEY)` 被多次刻意插入，同时同一段大写密文和变形 URL 也重复出现。

决定性步骤是把 `FREMEN` 识别为 Vigenère 密钥，对字幕中的普通可逆密文解码。视频只是引导载体，因此最终按表示层密码归入 Crypto。

## 解题过程

### 从字幕提取异常文本

直接查看 `fremen_filmbook.srt`，可见重复内容：

```text
YYI MRFBVV XMRX NMFLVS KLQ WGTIC AJ GMV WMRQBFVYW:
uykte://cbzky.ni/3pX-H-NspUZ
```

附近多次出现 `The Fremen (ARE KEY)`。括号内容不是自然字幕，组合后明确提示
`FREMEN ARE KEY`，即使用 `FREMEN` 作为密钥。

### Vigenère 解密

仅对字母推进密钥位置，保留空格、标点、数字、斜杠和连字符。大小写分别以
`A`/`a` 为基准：

```python
def vigenere_decrypt(ciphertext, key):
    out = []
    key_pos = 0

    for ch in ciphertext:
        if ch.isalpha():
            base = ord("A") if ch.isupper() else ord("a")
            shift = ord(key[key_pos % len(key)].upper()) - ord("A")
            out.append(chr((ord(ch) - base - shift) % 26 + base))
            key_pos += 1
        else:
            out.append(ch)

    return "".join(out)
```

以 `FREMEN` 解密得到：

```text
THE ANSWER LIES WITHIN THE STORY OF THE SANDWORMS:
https://youtu.be/3cS-Q-JglHU
```

链接目标提供的关键答案文本是 `one step closer to desert power`。这条信息已经写入正文，因此即使外链失效也不会丢失解题主线；保留
[原始视频链接](https://youtu.be/3cS-Q-JglHU) 仅用于核对题目链路。

### 按 flag 格式规范化

将短语转为小写、空格改为下划线，并按视频给出的 leetspeak 形式提交：

```text
UMDCTF{0n3_573p_cl053r_70_d353r7_p0w3r}
```

## 方法总结

- 核心技巧：从字幕中的重复异常文本和 `ARE KEY` 提示识别 Vigenère 密钥，连同被加密的 URL 一起解码。
- 识别信号：自然语言字幕中反复出现同一段高熵大写文本，且附近刻意强调某个单词“是密钥”，应先验证古典多表替换。
- 复用要点：Vigenère 实现应只在字母上推进密钥，否则 URL 中的标点会造成后半段错位；外链给出的最终关键信息必须同步记录在 WP 中。
