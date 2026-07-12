# shellgen2

## 题目简述

本题是 `shellgen` 系列的算法版，目标是程序化生成无字母数字 PHP webshell，尽量缩短生成代码长度。题目不主要考 Web 漏洞，而是考如何用 PHP 中可用的非字母数字表达式构造目标字符串。

题目附件是 Misc 版 PHP webshell 生成器，目标是在受限字符集下生成等价 PHP 表达式。官方 WP 给出三层思路：逐字符 `++` 暴力构造；用最长不下降子序列减少递增次数；预扫描 `a-z` 建索引并按字符频率设计 `hash_char`，让生成目标字符串时每个字符只需一次索引引用。

## 解题过程

题目源码见 [`frankli0324/d3ctf-shellgen` 的 misc 部分](https://github.com/frankli0324/d3ctf-shellgen/tree/master/misc)。下文保留的是生成器的三层算法：逐字符递增、最长不下降子序列优化和字符索引表。

这题的核心问题是：无字母数字 webshell 怎么写更短，能否程序化生成。题目偏算法，判定重点是生成出的 PHP 代码长度和可用性。

题目 WAF 的精确定义是：输出必须以 `<?php` 开头，之后每个字节只能来自 `0-9$_;+[].<?=>`，并且生成的 PHP 执行结果必须等于输入的随机小写字符串：

```python
def waf(phpshell):
    if not phpshell.startswith(b"<?php"):
        return False
    phpshell = phpshell[6:]
    for c in phpshell:
        if c not in b"0-9$_;+[].<?=>":
            return False
    return True
```

第一层是逐字符 `++`。

先构造出 `a`，再对每个目标字符逐个递增并拼接，这是最原始的暴力解法：

```python
target = input()
print("<?php")
print("$__=[].[];$__=$__[0.9+0.9+0.9+0.9];", end="")
for c in target:
    print("$_=$__;", end="")
    print("$_++;" * (ord(c) - ord("a") - 1), end="")
    print("$_1.=++$_;")
```

第二层是最长不下降子序列优化。

取 `target` 中所有最长不下降子序列，例如：

```text
acdbecfcdgef
a  b c cd ef
 cd e f  g
```

得到多条不下降序列后，根据原始顺序依次从各序列头部取字符输出。这里还涉及变量名如何分配的问题，因为变量数量和命名长度同样会影响最终 PHP 代码长度；题目使用的就是这一类优化算法。

第三层是预扫描字符表。

```python
print("<?php")
print("$_=[].[];$_=$_[0.9+0.9+0.9+0.9];", end="")

def hash_char(c):
    # 组成由 0 与 9 构造的索引，例如 0, 9, 90, 99, 900...
    return f(c)

target = input()

print("$__=[];")
for c in range(ord("a"), ord(max(target)) + 1):
    print("$_++;", end="")
    if c in target:
        print(f"$__[{hash_char(c)}]=$_;", end="")

print(f"$_=$__[{hash_char(target[0])}];")
for c in target[1:]:
    print(f"$_.=$__[{hash_char(c)}];")
print("?><?=$_;")
```

上面这个算法生成 PHP 代码的长度很大程度上取决于 `hash_char`。
这样一来，只需要对 `a-z` 的空间扫描一次，就能拿到所有字母的索引；生成目标字符串时，每个字母只需要一个语句。

实际上，可以依据目标字符串中字符出现频率排序，直接把更短的索引分配给更高频字符。更好的算法本质上都围绕两件事优化：减少 `++` 次数，减少新变量个数。

## 方法总结

- 核心技巧：把无字母数字 webshell 生成转化为字符串构造优化问题，用递增、序列分解和索引表减少 PHP 代码长度。
- 识别信号：题目限制字符集但允许 PHP 自增和数组/字符串下标，目标是生成指定 payload 而非绕过服务端漏洞。
- 复用要点：先构造一个一定可行的暴力生成器，再优化最长不下降序列、变量复用和字符索引；这种题的关键指标通常是输出长度和稳定通过率。

