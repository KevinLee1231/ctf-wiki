# L3akCTF 2025 babyrev Writeup

## 题目简述

题目给出一个简单的 64 位 ELF。程序要求输入 flag，随后打印成功或失败信息。检查二进制中的常量可以发现一段形似 flag 的密文：

```text
L3AK{ngx_qkt_fgz_ugffq_uxtll_dt}
```

程序没有内存破坏漏洞；获取 flag 的关键是还原小写字母替换表，因此按 Reverse 归档。

## 解题过程

### 还原比较逻辑

程序先初始化一个 128 字节映射表。非小写 ASCII 字符保持不变，26 个小写字母按固定规则替换：

```text
abcdefghijklmnopqrstuvwxyz
qwertyuiopasdfghjklzxcvbnm
```

读取输入后，程序逐字节执行 `input[i] = remap[input[i]]`，再将变换后的结果与内置密文比较。因此需要求的是上表的逆映射，而不是再次对密文应用正向映射。

程序还注册了 `SIGINT` 处理器：按下 `Ctrl+C` 会随机交换两个映射项并输出 `[interrupt remap]`。这只是干扰项；正常运行时映射始终固定，求解过程不需要触发信号。

### 逆向替换

可以从正向字符表直接构造逆表：

```python
encrypted = "L3AK{ngx_qkt_fgz_ugffq_uxtll_dt}"
plain_alphabet = "abcdefghijklmnopqrstuvwxyz"
cipher_alphabet = "qwertyuiopasdfghjklzxcvbnm"

reverse = dict(zip(cipher_alphabet, plain_alphabet))
flag = "".join(reverse.get(ch, ch) for ch in encrypted)
print(flag)
```

脚本仅替换小写字母，保留大写前缀、数字、下划线和花括号。运行结果为：

```text
L3AK{you_are_not_gonna_guess_me}
```

把该字符串输入题目程序后，经过正向映射恰好得到内置密文，程序输出 `YEEY!`。

## 方法总结

这道题考查最基础的静态数据流分析：输入并不是直接与常量比较，而是先经过一张单表替换。只要明确映射方向，解法就是对密文字母查逆表。

分析替换密码时，建议把明文字母表和密文字母表并排写出，再由代码构造逆映射，这比手工逐字符替换更不容易出错。信号处理器虽然会修改映射，但只有主动触发 `SIGINT` 才生效，不应把不会进入正常验证路径的逻辑误当成必要步骤。
