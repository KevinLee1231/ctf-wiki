# Rusted

## 题目简述

程序使用 Rust 的 `DefaultHasher` 分别计算输入中每个字符的哈希，并给出目标 flag 各字符对应的哈希序列。哈希不能直接求逆，但 flag 字符集很小，可以构造字符到哈希值的字典。

## 解题过程

源码中的核心逻辑等价于：

```rust
for c in guess.chars() {
    println!("letter of guess {}", calculate_hash(&c));
}
```

需要注意，必须复用题目相同的 Rust 类型和 `DefaultHasher` 实现，不能用其他语言中同名或自带的哈希函数代替。枚举常见 flag 字符集，把每个字符送入程序，记录输出并建立反向映射：

```python
alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_!?"
# 对每个字符调用题目程序，解析其 `letter of guess` 输出：
# reverse_map[hash_value] = character
# 最后逐项映射 enc.txt 中的目标值。
```

按 `enc.txt` 顺序查表后得到：

```text
n00bz{ru57_r3v3rs1ng_bu7_n0t_j41l_ch4ll_4s_r3qu3st3d_by_Qua5ar}
```

## 方法总结

单字符哈希虽然不可逆，却会因输入空间很小而退化为查表题。复现时最重要的是保持语言、字符类型和哈希实现一致，同时不要只枚举官方脚本中的小写字符集，否则会漏掉 flag 中的大写字母。
