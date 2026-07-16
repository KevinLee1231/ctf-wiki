# Shuffle！Shuffle！

## 题目简述

附件是 64 位 Windows PE。`.data` 中的全局 `seed` 初值为 `0x114514`，`main` 调用 `srand(seed)`，读取候选字符串后按其实际长度执行 Fisher–Yates 洗牌，最后与内置的 44 字节乱序字符串比较。二进制中虽然另有一个把 `seed` 写成 `0x666` 的 `runtime_env` 函数，但 `.CRT` 初始化表没有引用它，该函数不会在正常启动路径执行。

```text
23-64bed6}-xm5300-{faGa34-0e04c2e7c2a78f39a4
```

洗牌循环为：

```c
for (int i = length - 1; i > 0; --i) {
    int j = rand() % (i + 1);
    char temp = text[i];
    text[i] = text[j];
    text[j] = temp;
}
```

固定种子意味着每次运行都使用同一置换，但直接模拟时还必须复现题目使用的 Windows C 运行库 `rand()`，不能把 Linux glibc 的随机序列直接代入。

## 解题过程

一种方法是在相同 Windows CRT 下以种子 `0x114514` 生成交换序列，再逆置换密文。更稳妥的方法不依赖随机数实现：输入 44 个互不重复的字符，在调试器中于 `shuffle` 返回后读取被打乱的缓冲区，由字符位置直接恢复置换。

本题使用的探针明文及其洗牌结果为：

```text
probe:
kL9f2hEwR0xB8YpQvNjOtCz1Dg5sV3UaH4MbrX7iAqS+

shuffled_probe:
Nbgz45vH3+UL2wMj8tE0x97DCQphksVAa1XqfiSRYBrO
```

因为探针字符全部互异，若 `probe[i]` 出现在 `shuffled_probe[j]`，就说明原字符串第 `i` 个字符会被搬到乱序结果第 `j` 位；因此原 flag 的第 `i` 位就是密文的第 `j` 位。

```python
ciphertext = "23-64bed6}-xm5300-{faGa34-0e04c2e7c2a78f39a4"
probe = "kL9f2hEwR0xB8YpQvNjOtCz1Dg5sV3UaH4MbrX7iAqS+"
shuffled_probe = "Nbgz45vH3+UL2wMj8tE0x97DCQphksVAa1XqfiSRYBrO"

assert len(ciphertext) == len(probe) == len(shuffled_probe) == 44
assert len(set(probe)) == 44
assert set(probe) == set(shuffled_probe)

output_position = {
    character: index
    for index, character in enumerate(shuffled_probe)
}
flag = "".join(ciphertext[output_position[character]] for character in probe)
print(flag)
```

输出为：

```text
0xGame{5ffa9030-e204-4673-b4c6-ed433aca7228}
```

## 方法总结

固定随机洗牌本质上只是一个固定置换。若随机数运行库已知，可以重放 Fisher–Yates；若跨平台 `rand()` 不一致，则用等长、字符互异的 chosen plaintext 直接标记每个位置。本题必须保留 44 字节密文、探针输入和探针输出，三者足以在没有原程序和外部网站的情况下复现 flag。
