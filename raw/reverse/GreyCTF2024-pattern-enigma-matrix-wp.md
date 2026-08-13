# Pattern Enigma Matrix

## 题目简述

程序把 flag 交给 11 个编译期正则表达式校验。大部分正文可由前缀、总长度、分段格式和若干固定子串直接拼出；最后只剩 4 个取自 `d/e/f` 的字符与 2 个十六进制字符，共 $3^4\times16^2=20736$ 种候选，用原二进制逐个验证即可。

## 解题过程

关键约束可整理为：

```text
^grey\{
^.{68}$
[a-z0-9]{7}_[a-z0-9]{4}_[a-z0-9]{7}_[a-zG0-9]{8}_[a-f0-9]{32}
[def]{4}669870fc2fec[a-f0-9]{16}
ef6f33a17d0b9c\}$
```

散落的 `c0mp`、`1le`、`t1m`、`1m3`、`m4tch`、`1nG` 又确定前四段为：

```text
c0mp1le_t1m3_p4tt3rn_m4tch1nG_
```

32 位十六进制尾部中，正则固定了中间 12 位和最后 14 位，只需枚举开头 4 位及尚缺的 2 位。下面的脚本不猜测校验器内部状态，而是把候选交回题目二进制，以退出码作为最终判据：

```python
import itertools
import string
import subprocess

head = "grey{c0mp1le_t1m3_p4tt3rn_m4tch1nG_"
for a in itertools.product("def", repeat=4):
    for b in itertools.product(string.hexdigits[:16], repeat=2):
        tail = "".join(a) + "669870fc2fec" + "".join(b) + "ef6f33a17d0b9c"
        flag = head + tail + "}"
        p = subprocess.run(["./a_stripped.out", flag],
                           stdout=subprocess.DEVNULL)
        if p.returncode == 0:
            print(flag)
            raise SystemExit
```

唯一通过全部约束的候选是：

```text
grey{c0mp1le_t1m3_p4tt3rn_m4tch1nG_eefd669870fc2fec5cef6f33a17d0b9c}
```

## 方法总结

面对多正则校验，先做约束传播，再暴力搜索未定字符。把已知前缀、分隔符、固定后缀和字符集逐层合并，搜索空间会从不可行的整串枚举缩到两万余次；最终仍以原程序验证，避免手工理解 CTRE 生成的大量模板代码。
