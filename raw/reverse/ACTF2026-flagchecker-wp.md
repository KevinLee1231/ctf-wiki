# flagchecker

## 题目简述

题目是 LoongArch64 Go 静态程序，Go 符号恢复被刻意破坏。真实校验链分为反射入口、8 段并行 Feistel 变换和 LoongArch64 shellcode 中的 SM4 校验；大量蜜罐方法和反调试逻辑只用于干扰分析。

## 解题过程

附件是静态链接、去符号的 LoongArch64 ELF。`.gopclntab` magic 被篡改，常规 Go 符号恢复工具会失败；修复 magic 后可以恢复出函数符号，其中关键函数为：

```text
main.(*JGqVVFpm).CT77IKGJ
```

程序不会直接调用固定校验函数，而是根据输入前缀派生方法名，再通过 `reflect` 在 `Verifier` 上查找。派生逻辑为：

```text
method = upper(base32(sha256(prefix)[:5])[:8])
```

`prefix` 固定为 `ACTF{`，计算后得到：

```text
CT77IKGJ
```

因此这是唯一真实入口。其余大量方法会返回假 flag、死循环或退出进程。

题目限制 flag 格式为：

```text
ACTF{[0-9a-f]{32}}
```

真实方法接收中间 32 个 hex 字符，并按 4 字节切成 8 段 `seg[0..7]`。第 `i` 段的 key 依赖另一段明文：

```text
key[i] = HMAC-SHA256(masterKey, seg[(i + 3) mod 8])[:4]
out[i] = Feistel4(seg[i], key[i])
```

依赖关系形成长度为 8 的环。因为字符集固定为 hex，每段 4 字节的搜索空间为 $16^4=65536$。枚举 `seg0` 后，沿依赖链逆推：

```text
seg0 -> seg5 -> seg2 -> seg7 -> seg4 -> seg1 -> seg6 -> seg3
```

最后再用 `seg3` 派生回 `seg0` 的 key，检查闭合条件。

第二层输出还会进入 LoongArch64 shellcode。Go 侧只是：

```text
mmap 可写内存
拷入 shellcode.bin
mprotect 为可执行
通过汇编桥调用 shellcode
```

shellcode 有两类干扰：

```text
读取 /proc/self/status 检查 TracerPid
使用 clock_gettime 检查执行耗时
```

还插入了垃圾字节扰乱线性反汇编。核心逻辑是：使用固定 16 字节 key 对 32 字节输入做 SM4-ECB 加密，再与内置 `expectedCipher` 比较。`SM4` 的 S-box、FK、CK 常量完整保留，可以直接识别算法。

求解脚本需要的三个关键常量为：

```python
MASTER_KEY = bytes.fromhex(
    "447261676f6e41627973734c6f6f6e67"
    "36345265766572736543544632303236"
)

SM4_KEY = bytes.fromhex("4c6f6f6e67417263683634534d342121")

EXPECTED_CIPHER = bytes.fromhex(
    "47431c8eca012e7a3a6ff6c3807a168d"
    "e2ed5b5b8026866fc8ba4460320308fe"
)
```

先对目标密文做 SM4-ECB 解密，得到第一层目标输出：

```text
018f4771d52fb3b970bcdace9ef3e24bc1926f9bb054537ef00ba5e1efda4801
```

再用上面的 Feistel 环逆推 8 段 hex。满足所有约束的结果唯一：

```text
ACTF{fce553ec44532f11ff209e1213c92acd}
```

脚本运行后直接打印 `ACTF{...}`，说明校验器逆向和约束求解结果正确。

## 方法总结

- 核心技巧：修复 Go 符号恢复后定位反射真实入口，再把 shellcode 中的 SM4 层和 Go 中的 Feistel 环分开求逆。
- 识别信号：Go 静态程序 `.gopclntab` 异常、反射分发、架构 shellcode 和固定 flag 字符集同时出现时，应先找真实入口和可逆约束。
- 复用要点：不要被蜜罐方法带偏；先逆 shellcode 得到中间目标，再用 hex 字符集约束降低 Feistel 环的搜索空间。
