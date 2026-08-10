# 神秘的电话

## 题目简述

附件包含一段 Base64 文本和 `morse.wav`。Base64 解码后的提示是：“只有倒着翻过十八层的篱笆才能抵达北欧神话的终点”。这句话依次给出逆序、18 栏栅栏密码和维吉尼亚密码三个步骤；“北欧神话的终点”指向 Vidar-Team 名称来源 Víðarr，因此最后一层的密钥为 `vidar`。

## 解题过程

### 从音频恢复第一层密文

先解码 `encrypted_message.txt`：

```python
import base64

data = open("encrypted_message.txt", "rb").read()
print(base64.b64decode(data).decode())
```

在 Audacity 中观察 `morse.wav`：短信号是点，长信号是划，较长静音是字符分隔。手工转写或使用 `morse2ascii` 后，清理工具额外插入的下划线，得到：

```text
0223e_priibly__honwa_jmgh_fgkcqaoqtmfr
```

这个过程只依赖信号时长，原 PDF 中的波形截图没有额外信息，因此将结果直接转写为文本即可。

### 按提示逐层解密

提示中的三个关键词要按顺序应用：

1. “倒着”：将整段字符串逆序；
2. “十八层的篱笆”：进行 18 栏 Rail Fence 解密；
3. “北欧神话的终点”：用密钥 `vidar` 进行 Vigenère 解密。

前两步完成后的中间结果为：

```text
rmocfhm_wo_ybipe2023_ril_hnajg_katfqqg
```

参赛者的[赛后复盘](https://www.cnblogs.com/Mar10/p/17063246.html)也记录了相同的 Morse 文本、中间结果与密钥，可用于复核人工听写。维吉尼亚解密后得到：

```text
hgame{welcome_to_hgame2023_and_enjoy_hacking}
```

## 方法总结

- 核心技巧：先从音频时长还原 Morse 字符，再把自然语言提示映射为逆序、Rail Fence 和 Vigenère 三层古典密码。
- 易错点：`morse2ascii` 可能插入多余下划线；18 表示栅栏数；Vigenère 密钥是 `vidar`，不是泛指“诸神黄昏”的其它拼写。
- 复用要点：套娃题应保存每层中间结果，并利用下一层输出是否呈现自然语言结构来验证顺序和参数。
