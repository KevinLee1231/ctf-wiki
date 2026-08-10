# Chinese Character Encryption

## 题目简述

附件包含多行看似随机的汉字。每个汉字都独立映射到一个 ASCII 字符：先用 `pypinyin` 取得带数字声调的拼音，把拼音中所有字符的 ASCII 码相加，再对 128 取模。每一行都是同一明文的不同随机编码，因此解任意一行即可。

## 解题过程

以第一行密文为例：

```text
陉萏俦蘭貑謠祥冄剏髯簧凰蕆秉僦笆鼣雔耿睺渺仦殣櫤鄽偟壮褃劳充迧蝔镁樷萾懴雈踺猳钔緲螩蝒醢徣纒漐
```

对汉字 `陉`，`Style.TONE3` 给出 `xing2`，于是对应字节为：

$$
(\operatorname{ord}(x)+\operatorname{ord}(i)+\operatorname{ord}(n)+\operatorname{ord}(g)+\operatorname{ord}(2))\bmod128
$$

其他汉字逐个执行同一运算。完整脚本如下：

```python
from pypinyin import Style, lazy_pinyin

ciphertext = (
    "陉萏俦蘭貑謠祥冄剏髯簧凰蕆秉僦笆鼣雔耿睺渺仦殣櫤鄽偟壮褃"
    "劳充迧蝔镁樷萾懴雈踺猳钔緲螩蝒醢徣纒漐"
)

pinyin_list = lazy_pinyin(ciphertext, style=Style.TONE3)
plaintext = "".join(
    chr(sum(ord(character) for character in syllable) % 128)
    for syllable in pinyin_list
)

print(plaintext)
```

输出为：

```text
hgame{It*sEEMS|thaT~YOu=LEArn@PinYiN^VerY-WelL}
```

官方 PDF 只给出解码函数，没有把附件密文嵌进文档。第一行密文由当年参赛者题解补回，并已用同一 `TONE3` 规则独立计算验证：[HGAME Crypto WEEK1-4](https://www.cnblogs.com/sCh3n/p/15917384.html)。

## 方法总结

大量汉字并不意味着题目使用了复杂的中文密码学；关键是从提示中的 `pypinyin` 识别确定性映射。由于每个密文字独立产生一个 7 位结果，解码无需比较多行或恢复随机选择过程。复现时必须固定声调样式为 `TONE3`，否则数字声调缺失会改变 ASCII 求和结果。
