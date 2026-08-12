# DownUnderCTF 2022 Comes from a Land Down Under Writeup

## 题目简述

附件 `mate.aussie` 看起来整篇上下颠倒，实际是用 Unicode 倒置字符书写的 Aussie++ 程序。恢复可读源码后，程序会把多段字符串拼成一个小写 Base32 文本，但默认不打印关键变量。目标是还原脚本方向、补上输出语句并解码结果。

## 解题过程

先把整个文件的行顺序反转，再把每一行的字符顺序反转，最后按倒置字符对换，例如 `ɐ↔a`、`ǝ↔e`、`ƃ↔g`、`∀↔A`、`Ɛ↔3`。核心形式为：

```python
lines = open('mate.aussie', encoding='utf-8').read().splitlines()
upright = '\n'.join(line[::-1] for line in lines[::-1])
upright = ''.join(flip_table.get(ch, ch) for ch in upright)
open('mate-readable.aussie', 'w', encoding='utf-8').write(upright)
```

部分大写倒置字符不是一一对应，恢复结果可能丢失大小写，但 Aussie++ 的关键控制流仍清楚可读。程序定义了 `drunk_speak`、`a_pint`、`hiccup` 等函数，把返回片段依次拼到 `a_hiccup`，最后赋给：

```text
I RECKON i_should_print_the_flag = a_hiccup;
```

源码随后只打印醉酒提示，没有打印该变量。可在 Aussie++ 解释器中运行恢复后的代码，并在结尾加入：

```text
GIMME i_should_print_the_flag;
```

也可以不先翻转全文，直接把对应的倒置语句放到原脚本的倒置结尾。程序输出：

```text
irkugvcgppdjbruwknjw5yuiqbp4tok7nzpxhruwzgs4vb27itriraggsde3sx2o4ockhrugl7rirkjq4kcyix7cqszes7i=
```

源码中的“Sixty four? So we divide that by two”和“all lowercase”分别提示 Base32 与小写字母表。标准 Base32 不区分这里的字母大小写，把文本转成大写后解码即可：

```python
import base64

encoded = 'irkugvcgppdjbruwknjw5yuiqbp4tok7nzpxhruwzgs4vb27itriraggsde3sx2o4ockhrugl7rirkjq4kcyix7cqszes7i='
print(base64.b32decode(encoded.upper()).decode())
```

得到：

```text
DUCTF{ƐƖSSn∀_ɹ_n_sƖɥʇ_D∀Ɛɹ_NㄣƆ_∩0⅄_ℲI}
```

flag 内部的倒置 Unicode 字符是答案本身，不能再翻转成普通英文。

## 方法总结

本题包含两层表示：外层把完整 Aussie++ 源码在空间和字符层面倒置，内层程序再生成小写 Base32。先恢复足以理解控制流的源码，找到最终变量并补输出，比试图静态手工拼接所有醉酒函数更稳；得到编码串后再依据“64/2”和小写提示进入 Base32。最终 Unicode flag 应按解码字节原样保留。
