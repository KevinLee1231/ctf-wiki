# unpack

## 题目简述

附件是一个被修改过节区名称的 UPX-like ELF。自动脱壳工具无法按签名识别，但壳的控制流仍会在解压后跳回正常的 ELF 入口。找到 OEP 后即可直接分析校验函数：它把输入每一位加上索引，再与固定字节表比较。

## 解题过程

在调试器中从入口跟踪解压 stub。接近尾部时会恢复寄存器并跳到解压后的代码；题目样本的 OEP 为 `0x400890`，该处出现标准 x86-64 `_start` 形态：

```asm
xor     ebp, ebp
mov     r9, rdx
pop     rsi
mov     rdx, rsp
and     rsp, 0xfffffffffffffff0
```

在 OEP 暂停后转储进程映像，并按原 ELF 的加载段修复文件偏移；若导入表被壳破坏，再根据运行时解析结果重建。实际上本题也可直接从内存中的解压代码继续分析，不必为了得到 flag 完整修复可执行文件。

主校验循环等价于：

```c
for (i = 0; i <= 41; ++i) {
    if (input[i] + i != table[i])
        failed = 1;
}
```

固定表转写为十六进制后，逐字节减去索引即可：

```python
encoded = bytes.fromhex(
    "6868637069805B7578496D76757B756E4184716544824A858C"
    "827D7A824D907E92549888969857958FA6"
)
flag = "".join(chr(value - index) for index, value in enumerate(encoded))
print(flag)
```

输出：

```text
hgame{Unp@cking_1s_R0m4ntic_f0r_r3vers1ng}
```

## 方法总结

- 核心技巧：不要依赖节区名判断 UPX；跟踪解压尾跳并用标准 `_start` 指令序列确认 OEP。
- 识别信号：入口处存在大段自修改、解压循环和跨区跳转，而内存中随后出现完整 ELF 代码，通常说明存在运行时脱壳机会。
- 复用要点：完整重建 ELF 与恢复校验算法是两个目标；若只需解题，可在 OEP 后直接分析内存，减少不必要的修复工作。

> 原 PDF 只给出简短脱壳提示；公开复盘补足了 OEP 和校验常量，本文已重新计算明文。参考：[HGame 第二周 RE/PWN 复盘](https://blog.51cto.com/u_14601424/6839391)。
