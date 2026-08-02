# hulksmash

## 题目简述

题目使用 AES-ECB 加密 flag，16 字节密钥来自一串“键盘乱按”字符。附件同时给出多行 keysmash 样本。虽然单行看起来杂乱，但生成方式只会从左右手各自负责的一小组按键中取字符，并保留交替输入的结构；密钥空间远小于 16 个任意字节。

## 解题过程

观察第一行样本的奇偶位置，可以分离出左右手使用的字符集合。其余样本则揭示这些集合在不同片段中的排列方式。官方解法据此构造四组可能字符，对各组做排列，再把左右手字符交错组合成 16 字节候选密钥。

筛选不需要额外 oracle：用每个候选解密给定 AES-ECB 密文，检查 PKCS#7 填充与 `tjctf{` 前缀即可。

```python
from itertools import permutations
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

def interleave(left, right):
    return bytes(v for pair in zip(left, right) for v in pair)

def try_keys(ciphertext, left_sets, right_sets):
    for left_a in permutations(left_sets[0]):
        for right_a in permutations(right_sets[0]):
            for left_b in permutations(left_sets[1]):
                for right_b in permutations(right_sets[1]):
                    key = interleave(left_a + left_b, right_a + right_b)
                    if len(key) != 16:
                        continue
                    plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
                    try:
                        plaintext = unpad(plaintext, 16)
                    except ValueError:
                        continue
                    if plaintext.startswith(b"tjctf{"):
                        return key, plaintext
    raise ValueError("key not found")
```

实际使用的字符集合由附件样本直接提取，而不是预先假定某种键盘布局。遍历后得到明文：

```text
tjctf{low_entropy_keysmashuiyf8sa8uDYF987&^*&^}
```

## 方法总结

- 人类“随机敲键盘”通常只覆盖手指附近的小区域，并带有左右手交替、固定顺序等强结构，不能作为密码密钥来源。
- 多份样本的价值在于恢复生成规则；只统计字符频率而忽略位置关系，会留下大得多的搜索空间。
- AES 本身没有被攻破。攻击目标是低熵密钥生成器，`tjctf{` 前缀和合法填充则提供廉价、可靠的离线判定条件。
