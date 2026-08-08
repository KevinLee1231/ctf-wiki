# MiniLCTF2020 - Khronos

## 题目简述

Java 层只负责收集输入，校验位于 `libnative-lib.so` 的 JNI 函数。程序先把 flag 主体每两个字符组成一个 16 位整数，送入基于反馈移位寄存器的 `khronos()`；每组只比较 8 位结果，因此会产生多个候选。最后再用种子 1331 的滚动哈希筛出唯一 flag。

## 解题过程

JNI 入口先约束：

```text
长度为 32
前缀为 minil{
末尾为 }
```

中间 26 字节按两字节一组，共 13 组。`khronos()` 使用掩码 `0x88880C92`，每轮取 `R & mask` 的奇偶校验位，再执行 32 位左移与反馈；累计 8 个反馈位形成一字节，外层共运行 32 次。结果依次与下表比较：

```python
god = [241, 183, 26, 82, 107, 73, 118,
       2, 193, 214, 78, 182, 224]
```

可以按每组独立爆破可打印字符：

```python
alphabet = b'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz1234567890@$_}'

candidates = []
for target in god:
    group = []
    for a in alphabet:
        for b in alphabet:
            if khronos((a << 8) | b) == target:
                group.append(bytes([a, b]).decode())
    candidates.append(group)
```

第一层只有 8 位约束，每组会留下十余个候选。结合语义可先固定 `minil{KhR0nOs_1S_m..._t1mE}`，再枚举剩余组合，并严格模拟 C++ `unsigned int` 的 32 位溢出：

```python
def secure(text: str) -> int:
    h = 0
    for ch in text.encode():
        h = (h * 1331 + ch) & 0xffffffff
    return h & 0x7fffffff

assert secure('minil{KhR0nOs_1S_m4SteR_0f_t1mE}') == 1929691002
```

最终 flag 为：

```text
minil{KhR0nOs_1S_m4SteR_0f_t1mE}
```

该字符串也能完整通过当前仓库中的 JNI 源码校验。

## 方法总结

当校验函数按固定分组且各组互不影响时，先局部爆破再用全局哈希筛选，远比枚举整个 flag 空间有效。重写 native 哈希时必须保留无符号整数位宽；Python 任意精度整数若不加 `& 0xffffffff`，结果会与 C++ 不同。
