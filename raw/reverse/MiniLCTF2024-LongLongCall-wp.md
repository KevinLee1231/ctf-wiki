# miniLCTF 2024 Long long call Writeup

## 题目简述

ELF 对控制流加入大量基于 `ret` 的垃圾跳转，并在末尾插入 `leave; ret`，使 IDA 把连续逻辑错误识别为许多嵌套函数。程序还检查 `LD_PRELOAD` 与 `/proc/self/status` 中的 `TracerPid`。去除干扰后，真正校验只是把 44 字节输入每两个一组做加法和异或。

## 解题过程

### 绕过反调试与控制流干扰

`LD_PRELOAD` 检测用于阻止 hook `sleep`，`TracerPid` 检测则识别调试器。静态分析时可直接 patch 两处分支，使其固定走正常路径；动态调试也可在比较前修改返回值。

混淆块通过伪造返回地址和 `ret` 串联真实基本块，最后的 `leave; ret` 又破坏反编译器的函数边界。可写 IDAPython 脚本把确定的 `ret` 链改成直接跳转并 NOP 垃圾指令；本题逻辑很短，也可以在通过 `scanf("%44s", ...)` 后单步跟踪到比较循环。

### 逆向两字节变换

真实加密函数为：

```c
for (int i = 0; i < 44; i += 2) {
    uint8_t x = input[i] + input[i + 1];
    input[i] ^= x;
    input[i + 1] ^= x;
}
```

加法按 8 位溢出。每一对只有 $2^{16}$ 种输入，可独立暴力求逆：

```python
enc = bytes.fromhex(
    "BBBFB9BEC3CCCEDC9E8F9D9BA78CD795"
    "B0ADBDB488AF92D0CFA1A392B7B4C99E"
    "94A7AEF0A199C0E3B4B4BFE3"
)

def decrypt_pair(a, b):
    for x in range(256):
        for y in range(256):
            s = (x + y) & 0xff
            if (x ^ s) == a and (y ^ s) == b:
                return x, y
    raise ValueError("no pair")

plain = bytearray()
for i in range(0, len(enc), 2):
    plain.extend(decrypt_pair(enc[i], enc[i + 1]))

print(plain.decode())
```

输出为：

```text
miniLCTF{just_s1mple_x0r_1n_lon9_l0ng_c@ll!}
```

也可用 44 个 8 位 BitVec 把同一方程交给 Z3；由于每组完全独立，枚举已经足够快且更直观。

## 方法总结

控制流混淆只增加了定位真实逻辑的成本，并没有提高校验算法强度。遇到 `ret` 链和错误函数边界时，应以运行时数据流为准，先找到输入缓冲区与最终比较，再决定静态修复还是动态跟踪。还原 8 位 C 运算时必须显式保留模 $2^8$ 溢出语义。
