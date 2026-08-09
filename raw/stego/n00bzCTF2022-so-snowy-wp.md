# so_snowy!

## 题目简述

附件文本使用 SNOW 空白隐写，并给出候选口令表。需要逐个口令调用 `stegsnow` 解密隐藏在行尾空格与制表符中的数据。

## 解题过程

对 `wordlists.txt` 中的每个候选口令运行解密，并只保留含 `n00bz{` 的输出：

```bash
while IFS= read -r password; do
    stegsnow -C -p "$password" enc.txt
done < wordlists.txt | grep 'n00bz{'
```

恢复结果为：

```text
n00bz{st3g_1s_s0_sn0wy}
```

## 方法总结

空白隐写对复制、自动格式化和行尾清理非常敏感，必须直接处理原附件。字典攻击时也应保留原始口令的一行一项边界，避免 shell 分词破坏包含特殊字符的候选值。
