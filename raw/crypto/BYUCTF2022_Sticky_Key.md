# BYUCTF 2022 - Sticky Key

## 题目简述

邮件由大量 `Å`、`∫`、`ç`、`` 等符号组成。苹果标志提示 Mac，题名 “Sticky Key” 则暗示输入时 Option（Alt）键一直处于按下状态。

## 解题过程

这不是通用 Unicode 替换，而是美式 Mac 键盘的 Option 键映射。仓库 `solve.py` 给出了完整、按位置一一对应的两行码表：

```python
alt = "ÅıÇÎ´Ï˝ÓˆÔÒÂ˜Ø∏Œ‰Íˇ¨◊„˛Á¸å∫ç∂´ƒ©˙ˆ∆˚¬µ˜øπœ®ß†¨√∑≈¥Ω¡™£¢∞§¶•ªº–≠⁄€‹›ﬁﬂ‡°·‚—±”’ ≥≤æ"
char = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz1234567890-=!@#$%^&*()_+{} .,'"
table = dict(zip(alt, char))
plain = ''.join(table.get(c, c) for c in message)
```

也可以只定位 flag。密文末段出现 `ç†ƒ`，在该键盘映射中恰为 `ctf`，因此从其前面的 `byu` 对应符号开始逐字转换即可。完整解码邮件说明键盘被饮料弄坏，并给出：

```text
byuctf{dont_leave_soda_by_your_keyboard}
```

题面要求全小写，按原样提交。

## 方法总结

特殊符号串的关键上下文是输入平台和修饰键。苹果标志、题名和可辨认的 `ctf` 模式共同确认了 Mac Option 键映射；不需要把它误判为复杂替换密码或字体隐写。
