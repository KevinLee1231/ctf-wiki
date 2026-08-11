# 欢迎参加 HGame！

## 题目简述

附件是一段 Base64 文本。解码后得到由点、横线和空格组成的摩尔斯电码，其中下划线也用一个独立的摩尔斯符号表示。决定性步骤是连续完成两层可逆表示转换，因此归入 Crypto。

## 解题过程

原始字符串为：

```text
Li0tIC4uLi0tIC4tLi4gLS4tLiAtLS0tLSAtLSAuIC4uLS0uLSAtIC0tLSAuLi0tLi0gLi4tLS0gLS0tLS0gLi4tLS0gLS0tLS0gLi4tLS4tIC4uLi4gLS0uIC4tIC0tIC4uLi0t
```

先做 Base64 解码：

```python
import base64

data = "Li0tIC4uLi0tIC4tLi4gLS4tLiAtLS0tLSAtLSAuIC4uLS0uLSAtIC0tLSAuLi0tLi0gLi4tLS0gLS0tLS0gLi4tLS0gLS0tLS0gLi4tLS4tIC4uLi4gLS0uIC4tIC0tIC4uLi0t"
print(base64.b64decode(data).decode())
```

得到：

```text
.-- ...-- .-.. -.-. ----- -- . ..--.- - --- ..--.- ..--- ----- ..--- ----- ..--.- .... --. .- -- ...--
```

按国际摩尔斯表逐组解码，`..--.-` 对应下划线 `_`，结果为：

```text
W3LC0ME_TO_2020_HGAM3
```

补上比赛规定的 flag 外壳：

```text
hgame{W3LC0ME_TO_2020_HGAM3}
```

## 方法总结

- 核心技巧：依据字符集和填充特征识别 Base64，再把解码结果按摩尔斯符号逐组转换。
- 识别信号：长字符串只含 Base64 字符；第一层结果只含 `.`、`-` 与分隔空格。
- 复用要点：`..--.-` 是下划线而不是空格；某些在线工具缺少该扩展符号，需要手工补回。
