# hidden

## 题目简述

程序表面上在构造 CRC32 表，但建表过程中会解码一段代码到可执行内存并间接调用，导致 IDA 在 `sub_140001010` 附近提示 `call analysis failed`。真正的 40 字节 flag 变换和比较都在这段运行时生成的代码中。

## 解题过程

从输入函数向下跟踪可知，程序要求长度恰好为 40，并把前后各 20 字节交给带 CRC32 外壳的函数。动态调试到可执行缓冲区，或在间接调用前转储内存，就能恢复隐藏校验逻辑：

```c
for (int i = 0; i < 19; i++) {
    for (int j = 0; j < 2; j++) {
        flag[i] ^= flag[19 + i];
        flag[i] += flag[38 + j];
        flag[19 + i] += 0x99;
        flag[19 + i] ^= flag[i];
    }
}
```

结果与五个 64 位常量逐字节比较：

```python
words = [
    0x7b754b47758f8846,
    0x48757a7b8a7f798e,
    0x4b7d87824b7b7b7b,
    0x81817350a79b885d,
    0x7d65574f57fa729a,
]
```

x86-64 为小端序，因此先把每个常量按 8 字节 little-endian 展开。`flag[38]` 和 `flag[39]` 没有在循环中被修改，可直接作为两轮加法使用的常量。逆运算必须把内层循环按 `j=1,0` 倒序执行：

```python
encrypted = bytearray().join(
    word.to_bytes(8, "little") for word in words
)
assert len(encrypted) == 40

plain = bytearray(encrypted)
tail = plain[38:40]

for i in range(19):
    left = plain[i]
    right = plain[19 + i]

    for j in (1, 0):
        right ^= left
        right = (right - 0x99) & 0xff
        left = (left - tail[j]) & 0xff
        left ^= right

    plain[i] = left
    plain[19 + i] = right

print(plain.decode())
```

输出：

```text
hgame{h1dden_1n_mem0ry_15_excited_2eeee}
```

## 方法总结

- 核心技巧：识别“CRC32 建表”只是运行时代码解码的载体，在间接调用处转储真实代码。
- 关键细节：五个整数要按小端序展开；字节加减需 `& 0xff`；内层两轮必须逆序撤销。
- 复用要点：静态反编译出现无法分析的间接调用时，应结合内存权限变化和动态执行地址判断是否存在 SMC/运行时解码。
