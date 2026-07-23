# UMDCTF2022 Go Away! Writeup

## 题目简述

附件是一个静态链接、已去符号的 64 位 IBM S/390 Go 可执行文件。它包含 Docker 环境检测和 AES-CFB 解密逻辑，很容易让人把精力放在跨架构运行或解密字符串上；真正的 flag 却由 Go 编译器写入的源文件路径泄露。

## 解题过程

先确认文件类型：

```text
ELF 64-bit MSB executable, IBM S/390, statically linked, Go BuildID, stripped
```

Go 二进制通常保留构建路径、运行时源码路径和其他调试字符串。直接搜索可打印字符串：

```bash
strings -a -n 8 go-away | grep -F UMDCTF
```

输出中出现：

```text
/home/birch/programming/UMDCTF-2022-Challenges/rev/go-away/UMDCTF{haha_not_that_easy_homie}/dev/source.go
```

路径的目录名就是官方 flag：

```text
UMDCTF{haha_not_that_easy_homie}
```

源码中的 AES-CFB 逻辑属于干扰项。密钥为 `abc&1*~#^2^#s0^=)^^7%b34`，IV 是固定 16 字节数组；三个 Base64 密文分别解出：

```text
/.dockerenv
lol hi ur getting close!
UMDCTF{g3t_l0st_h0m13}
```

主程序只打印第二项提示，第三项即使被解出也是诱饵，并非 `author-solve/flag.txt` 中记录的答案。以官方 flag 文件和二进制实际构建路径交叉验证后，应采用路径中泄露的 flag。

## 方法总结

跨架构和加密代码不一定是决定性障碍。对 Go、Rust 等包含丰富构建元数据的二进制，第一轮应先做 `file`、`strings` 和路径检索。发现多个候选 flag 时，不能看到 `UMDCTF{...}` 就停止；应结合执行路径、官方答案和产物来源判断哪些是诱饵，哪些是构建时意外泄露。
