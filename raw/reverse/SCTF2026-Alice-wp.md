# Alice

## 题目简述

附件是 macOS 应用。主程序校验输入 PNG 后加载 `libalice_stage0.dylib`；stage0 用 LCG 字节流异或解密 `Resources/core.dat`，把得到的 Mach-O dylib 写到临时目录并 `dlopen`；下一阶段再解密 `alice.dat`，最终留下一个 QR Code Reed–Solomon 块的约束。

恢复出的 QR 参数为 Version 5、纠错等级 M、mask 4。第一 RS 块含 43 个 data codeword 和 24 个 parity codeword。已知末尾 8 字节 padding 及全部 parity，未知的是前 35 字节；关键是把固定 RS 编码看作 $\mathrm{GF}(2)$ 上的线性映射，而非爆破 280 个未知 bit。

## 解题过程

### 1. 还原动态加载链

逆向 stage0 的 LCG 状态更新和逐字节 XOR，解出 `core.dat`；确认输出含合法 Mach-O header 后，再跟进导出函数对 `alice.dat` 的解密。最终可以提取第一 RS 块的 24 字节校验：

```text
4f f9 b2 3e 24 ac b5 98 69 bd cb af ab cc 1f aa
6e 01 2e c4 58 ce fd 61
```

43 个 data codeword 的末尾 8 字节固定为：

```text
ec 11 ec 11 ec 11 ec 11
```

### 2. 把 RS parity 写成二进制线性方程

QR 的 RS 运算位于 $\mathrm{GF}(256)$，本原多项式为 `0x11d`。加法是 XOR，而乘以固定域元素对一个字节的 8 个 bit 是 $\mathrm{GF}(2)$ 线性变换。

先令未知 35 字节全零、保留已知尾部，计算基准 parity。再逐个翻转 280 个未知 bit，每次重算 parity 并与基准异或，就得到该 bit 对 192 个输出 bit 的贡献。将目标 parity 与基准之差作为右端项，构造 192 条 $\mathrm{GF}(2)$ 方程。

### 3. 加入 QR byte mode 结构约束

仅靠 192 条 RS 方程不足以唯一决定 280 bit。Version 5 的 byte mode 数据结构提供额外约束：

```text
mode       = 0100
length     = 8-bit payload length
payload    = SCTF{...}
terminator = 至多 4 个 0
padding    = 字节对齐后交替 EC 11
```

枚举可能的 payload 长度，对每个长度加入 mode、长度字段、`SCTF{` 前缀、`}` 后缀、terminator 和 padding 方程，然后做 GF(2) 高斯消元。只有长度 29 的解同时满秩、全部可打印，并能重新生成完全一致的 24 字节 parity。

对应 data codeword 起始为：

```text
41 d5 34 35 44 67 b6 63 07 27 47 56 e6 55 f6 64 ...
```

按 byte mode 解析得到：

```text
SCTF{f0rtune_f@v0rs_th3_blod}
```

## 方法总结

本题前半段的多级 dylib 解密只是为了取得约束，真正求解点是“固定 RS 编码对输入 bit 线性”。用单 bit 差分自动生成线性矩阵，比手推 GF(256) 生成矩阵更不易出错；再把 QR 格式和 flag 形状写成同一组线性约束即可消除自由变量。最终必须重新执行 RS 编码和 QR 字段解析双重验证，不能只凭可打印字符串接受候选。
