# flimsy-fingered-latin-teacher

## 题目简述

题面给出字符串 `ykvyg}pp[djp,rtpelru[pdoyopm|`，并提示老师在 Dell 笔记本上盲打时手指没有放在正确的主键位。Dell 使用标准 QWERTY 布局，密文中的每个字符都是目标按键右侧相邻的键。

## 解题过程

建立每一排键盘字符的映射，把输入字符向左平移一格：

```python
rows = [
    "1234567890-=",
    "qwertyuiop[]\\",
    "asdfghjkl;'",
    "zxcvbnm,./",
]

def left_key(c):
    for row in rows:
        if c in row:
            return row[row.index(c) - 1]
    return c

cipher = "ykvyg}pp[djp,rtpelru[pdoyopm|"
print(''.join(left_key(c) for c in cipher))
```

例如 `y -> t`、`k -> j`、`v -> c`、`} -> {`、`[ -> p`。逐字符还原后得到：

```text
tjctf{oopshomerowkeyposition}
```

## 方法总结

- 键盘偏移题应根据提示先确认布局，再判断是整只手左移还是右移；本题密文按键位于明文右侧。
- 花括号、方括号和标点也必须纳入对应键盘行，不能只处理字母。
- 这种变换是物理布局替换，不是 Caesar 移位，按字符编码做固定加减会失败。
