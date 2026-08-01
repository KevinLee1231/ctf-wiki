# GlacierCTF 2025 repo-viewer-revenge

## 题目简述

加强版不再调用 lesspipe。服务先用 GNU tar 列出上传的 gzip tar，并拒绝列表中出现的符号链接或硬链接；随后另一个 Rust 程序使用 `tokio-tar` 解包，只接受路径 `README.md`，最后打开 `/tmp/README.md`。

漏洞来自两个 TAR 解析器对 PAX 扩展尺寸的不同理解。校验器和解包器在文件边界上失步，使攻击者能把一个指向 `/flag.txt` 的 `README.md` 符号链接藏进数据区。

## 解题过程

### 1. 明确两个解析器的安全边界

预检查近似为：

```bash
tar -tvzf upload.tar.gz | grep '^[lh]'
```

真正解包则由 Rust 的 `tokio-tar` 完成。安全性隐含依赖一个未经验证的假设：GNU tar 与 `tokio-tar` 会把完全相同的字节解释成完全相同的 entry 序列。只要这个假设不成立，前置检查就不是解包结果的可靠策略。

### 2. 利用 PAX 与 USTAR size 不一致

构造归档时，让一个普通条目同时具有两种尺寸：

- PAX 扩展头声明 `size=1024`；
- 紧随其后的传统 USTAR header 的 size 字段写成 `0`；
- 其 1024 字节“文件数据”的开头，放置另一个合法 TAR header；
- 内层 header 的路径为 `README.md`，类型为符号链接，目标为 `/flag.txt`。

GNU tar 正确地优先使用 PAX size，于是把后面的 1024 字节整体当成普通文件内容。预检查看到的 entry 列表中没有 link。

受影响的 `tokio-tar` 却按照 USTAR size 的 0 字节推进流位置，没有跳过那 1024 字节。它立即把数据区开头的内层 header 当成下一个外层 entry，于是解出攻击者隐藏的 `README.md -> /flag.txt`。

这就是 [TARmageddon / CVE-2025-62518](https://edera.dev/stories/tarmageddon) 的边界解析问题。披露文章确认了根因：PAX 声明真实大小 $X$、USTAR 声明 0 时，脆弱解析器按 0 前进，从而把嵌套 TAR 数据当成外层条目。本文已经概括了该外链对本题有用的完整机制。

### 3. 读取被走私的符号链接

仓库的 `repro_generator.cpp` 手工计算 PAX record 长度、USTAR checksum 和 512 字节对齐，生成 `pax_bug_compact.tar.gz`。将该归档按服务协议上传后：

1. GNU tar 的 link 检查通过；
2. Rust 解包器接受其所看到的路径 `README.md`；
3. `/tmp/README.md` 实际成为指向 `/flag.txt` 的符号链接；
4. 服务按正常逻辑打开 README，返回 flag。

源码实例结果为：

```text
gctf{Ru5t_m4k3s_3v3Ry7h1ng_5eCuR3_71a9f2ed8}
```

## 方法总结

本题是典型 parser differential：检查和使用不是同一个解析器，攻击者就会寻找两者对边界、规范化、扩展头或错误恢复的分歧。修复不能只是增加一条 GNU tar grep；应使用与解包相同且已修复的库完成校验，优先采用 PAX 元数据并验证尺寸一致性，解包后再检查真实文件类型和解析后的目标路径。Rust 的内存安全也无法消除这种格式语义漏洞。
