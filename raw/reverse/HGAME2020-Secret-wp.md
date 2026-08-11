# Secret

## 题目简述

程序用伪主流程、反调试、信号处理和父子进程配合隐藏真正校验逻辑。关键代码不是连续存在于普通函数中，而是由子进程接收并分阶段执行：先设置 XTEA 的 delta、密钥、密文和轮数，再完成比较。把这些运行时参数全部恢复后即可离线解密。

## 解题过程

静态入口中的大量分支并不是最终算法。动态调试时应跟随进程创建和 `sigaction` 安装，记录各处理函数实际写入的状态；若调试器只停留在初始进程，就会看到故意留下的假主流程。把分散阶段串起来后，核心算法是 32 轮 XTEA，参数为：

```python
DELTA = 0x9E3779B9
KEY = [
    0x42655F29,
    0x9E822EFC,
    0xDA278C92,
    0x4E355A62,
]
```

调试器中取得的 14 个密文字显示为：

```text
E9C8A927 B473A9BA F972C0AA 0080FAA3
D3C2F4D9 C56B3FFB 5ED9D3D3 771D9686
3FC500E6 B927BC98 ACC3AA09 2424DC6A
04E30506 778CE765
```

这些值按内存字节分组显示，而 XTEA 以 32 位整数运算。调整字节序后，实际送入每组解密的整数为：

```python
CIPHER = [
    0x27A9C8E9, 0xBAA973B4,
    0xAAC072F9, 0xA3FA8000,
    0xD9F4C2D3, 0xFB3F6BC5,
    0xD3D3D95E, 0x86961D77,
    0xE600C53F, 0x98BC27B9,
    0x09AAC3AC, 0x6ADC2424,
    0x0605E304, 0x65E78C77,
]
```

XTEA 解密要按加密的逆序更新 $v_1$、sum 和 $v_0$，所有运算保持在 32 位无符号整数内：

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9
KEY = [0x42655F29, 0x9E822EFC, 0xDA278C92, 0x4E355A62]
CIPHER = [
    0x27A9C8E9, 0xBAA973B4, 0xAAC072F9, 0xA3FA8000,
    0xD9F4C2D3, 0xFB3F6BC5, 0xD3D3D95E, 0x86961D77,
    0xE600C53F, 0x98BC27B9, 0x09AAC3AC, 0x6ADC2424,
    0x0605E304, 0x65E78C77,
]

def decrypt_pair(v0, v1):
    total = (DELTA * 32) & MASK
    for _ in range(32):
        mix = (((v0 << 4) & MASK) ^ (v0 >> 5))
        mix = (mix + v0) & MASK
        key_part = (total + KEY[(total >> 11) & 3]) & MASK
        v1 = (v1 - (mix ^ key_part)) & MASK

        total = (total - DELTA) & MASK

        mix = (((v1 << 4) & MASK) ^ (v1 >> 5))
        mix = (mix + v1) & MASK
        key_part = (total + KEY[total & 3]) & MASK
        v0 = (v0 - (mix ^ key_part)) & MASK
    return v0, v1

plain = bytearray()
for i in range(0, len(CIPHER), 2):
    v0, v1 = decrypt_pair(CIPHER[i], CIPHER[i + 1])
    plain += struct.pack("<II", v0, v1)

print(plain.decode())
```

输出为：

```text
hgame{No_0Ne_c4N_t0_Be_@_$taR.Th3y_st1lL_CAn_c4N_sH1n3.}
```

官方 PDF 只保留了部分动态分析过程；信号处理链、完整密钥和密文由 [HGAME 2020 Week 4 Reverse 复盘](https://www.cnblogs.com/harmonica11/p/12292804.html) 交叉核对。关键参数和可独立运行的解密逻辑已经写入正文，不依赖外链才能理解解法。

## 方法总结

- 遇到信号处理、父子进程和运行时下载代码时，分析单位应从“单个 main 函数”切换为跨进程状态变化和真实执行轨迹。
- XTEA 恢复中最常见的错误是密文字节序、轮内更新顺序以及 Python 大整数没有模拟 `uint32_t` 溢出。
- 将动态取得的 delta、key、轮数、密文和比较结果全部固化到离线脚本，才能形成可复现而非只依赖调试截图的题解。
