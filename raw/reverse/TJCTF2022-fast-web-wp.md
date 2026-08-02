# TJCTF2022 fast-web

## 题目简述

附件是一个修改过的 GoAhead C Web 服务器，`/flag.txt` 受 HTTP Basic Authentication 保护，用户名固定为 `sicer`。`auth.txt` 保存完整 SHA-512 十六进制摘要，但手写汇编校验器错误地把候选摘要中的零字当成比较结束标志，因此无需破解原密码，只需寻找满足短前缀条件的碰撞口令。

## 解题过程

`verify.s` 每次把存储摘要的四个十六进制字符解析成一个 16 位数，与候选 SHA-512 缓冲区中的一个字比较。比较成功后，它先把候选指针增加 2，再执行：

```asm
cmp WORD PTR [r9], 0
jnz cmphash.l1do
mov eax, 1
```

因此一旦候选摘要的下一个 16 位字为零，函数就提前返回成功。自定义 SHA-512 输出还以 64 位字的小端形式位于内存中。结合 `auth.txt` 开头的 `e8b8`，普通 SHA-512 摘要只需满足：

```text
digest[4:6] == 00 00
digest[6:8] == e8 b8
```

官方 C 求解器并行枚举可打印字符串，并以 `((uint32_t *)hash)[1] == 0xb8e80000` 检查这四个字节。可用口令 `2377455722` 的摘要以 `57cc19e20000e8b8...` 开头；这一具体口令也可由[独立复现记录](https://archive.jro.sg/writeups/rev/fast-web.html)交叉验证。用它进行 Basic Auth：

```bash
curl -u 'sicer:2377455722' 'http://target/flag.txt'
```

即可得到 `tjctf{g0_ah3ad_and_us3_a_n0rm4l_w3b_serv3r_pls_4b470205e474e398}`。

## 方法总结

问题不在 SHA-512 强度，而在摘要比较把二进制数据当作带终止符的字符串。逆向密码校验时要分别确认摘要的逻辑顺序、内存端序、比较粒度与终止条件；任何一项错误都可能把 512 位匹配降成很短的前缀搜索。本题实际只需约 $2^{32}$ 次哈希，并可通过多进程并行完成。
