# single

## 题目简述

题目给出一段使用单表替换密码处理的长文本。密文的字符分布和单词边界仍然存在，提示它不是压缩或现代加密，而是可以逐字符逆变换的古典密码。官方材料确认所用算法为 Atbash。

## 解题过程

Atbash 将字母表首尾对应：`A` 与 `Z`、`B` 与 `Y`，小写字母同理。该映射是自反的，因此加密和解密使用同一个变换：

```python
def atbash(text):
    out = []
    for ch in text:
        if "a" <= ch <= "z":
            out.append(chr(ord("z") - (ord(ch) - ord("a"))))
        elif "A" <= ch <= "Z":
            out.append(chr(ord("Z") - (ord(ch) - ord("A"))))
        else:
            out.append(ch)
    return "".join(out)
```

把题目密文传入函数后，在还原出的文本中找到：

```text
greyhats{we_love_cats_so_very_much}
```

## 方法总结

Atbash 不需要密钥，且不会改变标点、空格与重复模式。看到字母分布近似自然语言、非字母字符原样保留时，可以优先尝试 ROT、Caesar、Atbash 等基础单表替换；验证时应检查整段文本是否恢复为自然语言，而不只检查某个局部是否像 Flag。
