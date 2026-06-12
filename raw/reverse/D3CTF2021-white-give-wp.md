# white give

## 题目简述

本题是 LLVM pass 混淆后的逆向题，程序没有做控制流混淆，主要干扰来自全局变量 AES 加密、常数拆分和位运算表达式替换。验证逻辑要求输入 64 字节 flag，把每 4 字节做 SHA-256 后再经过 S-box、轮密钥异或和加法变换，与内置 `ffllaagg` 数据比较。

关键识别信号是表达式替换模式和全局变量访问前后的解密/回写行为：先还原常量和表达式，再按程序中的校验流程反推或调试获取正确输入。

## 解题过程

题目使用 LLVM pass 做了三类混淆，但没有做控制流混淆，因此重点是还原数据和表达式语义。

第一类是全局变量加密。程序在访问全局变量前用 AES 把数据解密到栈上，使用结束后再加密写回全局变量。调试时可以在解密后 dump 出真实的全局数据，例如 S-box、round key 相关数组和最终比较用的 `ffllaagg`。

第二类是常数加密。对 `Store`、`ICmp` 等指令中的常数 `z`，混淆 pass 会令 `z = x + y` 或 `z = x ^ y`，把 `x` 存在全局变量里，`y` 留作指令操作数。处理时可把相关全局变量标成 const，让 IDA 或反编译器自动折叠出原始常数。

第三类是 MBA（mixed boolean-arithmetic）表达式替换，把 `add/xor/or` 等简单运算改写成布尔运算和算术运算的等价式。例如：

```text
x + y = (x ^ y) + 2 * (x & y)
x ^ y = 2 * (x | y) - y - x
x | y = y + x - (x & y)
```

这类混淆模型可参考 Ninon Eyrolles 的博士论文《Obfuscation with Mixed Boolean-Arithmetic Expressions: reconstruction, analysis and simplification tools》：<https://tel.archives-ouvertes.fr/tel-01623849/document>。实际解题时不需要逐条手算，使用 Quarkslab 的 [`sspam`](https://github.com/quarkslab/sspam) 等工具可把表达式规约回原始语义，再结合动态 dump 恢复常量和 S-box。

还原后主校验流程如下：

```c
if (strFlag.size() != 64) {
    return 1;
}

for (size_t i = 0; i < 16; ++i) {
    sha256(flag + i * 4, 4, sha + 32 * i);
}

srand(0);
for (int i = 0; i < 256; ++i) {
    key[i] = rand();
}

for (size_t y = 0; y < 16; ++y) {
    for (size_t x = 0; x < 16; ++x) {
        mySBOX[16 * y + x] = 16 * (15 - y) + x;
    }
}

for (int index = 0; index < 2; ++index) {
    for (size_t i = 0; i < 16; ++i) {
        for (size_t t = 0; t < 256; ++t) {
            sha[t + index * 256] = mySBOX[sha[t + index * 256]];
        }
        build_round_key(roundKey, key + i * 16);
        for (size_t t = 0; t < 256; ++t) {
            sha[t + index * 256] ^= ((uint8_t *)roundKey)[t];
            sha[t + index * 256] += t;
        }
    }
}

return memcmp(sha, ffllaagg, 512);
```

由于每 4 字节 flag 独立进入 SHA-256，后续变换又是确定的 S-box、round key 异或和加法，恢复出 `ffllaagg` 后即可按块反推或调试枚举输入。题目真正要绕过的是 pass 混淆，不是复杂控制流。

## 方法总结

- 核心技巧：识别 LLVM pass 生成的常量加密、全局变量加密和代数等价表达式替换，先做表达式/常量恢复，再分析真实校验逻辑。
- 识别信号：大量形如 `x + y`、`x ^ y`、`x | y` 的复杂等价变形，以及访问全局变量前后出现 AES 解密和回写。
- 复用要点：没有控制流混淆时，不必一开始就上重型自动化；可以先让 IDA/调试器恢复常量，再用 sspam 等代数化简工具辅助清理位运算表达式。

