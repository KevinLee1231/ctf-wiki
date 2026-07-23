# Art

## 题目简述

附件同时给出 Linux 与 Windows 版本，程序带有标准 UPX 壳。脱壳后，主函数会读取 28 字节输入，保留一份原文用于哈希校验，再原地修改前 27 字节：

```c
for (i = 0; i <= 27; ++i)
    input_copy[i] = input[i];

for (i = 1; i <= 27; ++i)
    input[i - 1] ^=
        (input[i - 1] % 17 + input[i]) ^ 0x19;

if (!strcmp(input, enc) && sub_401550(input_copy))
    puts("Good job");
```

逐字节变换中同时出现取模、整数加法和异或。它对单个明文字节不是一一映射，不能像普通异或链那样直接写出唯一逆变换。反编译函数 `sub_401550` 是对原始输入的 SHA-1 校验，目的正是从多个局部方程解中选出唯一候选。

## 解题过程

先使用 `upx -d` 脱壳，再在反编译器中定位 `strcmp` 前的循环。记原始输入为 $x_0,x_1,\ldots,x_{27}$，比较数组为 $c_0,c_1,\ldots,c_{27}$。循环对应

$$
c_i=x_i\oplus\left((x_i\bmod 17+x_{i+1})\oplus 0x19\right),
\qquad 0\le i<27.
$$

最后一个字节没有被循环修改，因此

$$
x_{27}=c_{27}=0x7d,
$$

也就是字符 `}`。已知 $x_{i+1}$ 后，可以枚举可打印范围内的 $x_i$，保留满足第 $i$ 个方程的值，再继续向前搜索。由于映射并非单射，每一层可能出现多个分支，适合用 DFS，而不是在某一层任意取第一个解。

完整搜索脚本如下：

```python
CHECK = [
    0x02, 0x18, 0x0F, 0xF8, 0x19, 0x04, 0x27,
    0xD8, 0xEB, 0x00, 0x35, 0x48, 0x4D, 0x2A,
    0x45, 0x6B, 0x59, 0x2E, 0x43, 0x01, 0x18,
    0x5C, 0x09, 0x09, 0x09, 0x09, 0xB5, 0x7D,
]

candidate = [0] * len(CHECK)
candidate[-1] = CHECK[-1]
solutions = []


def dfs(pos):
    if pos < 0:
        solutions.append(bytes(candidate))
        return

    next_byte = candidate[pos + 1]
    for current in range(0x20, 0x7F):
        transformed = (
            current
            ^ ((current % 17 + next_byte) ^ 0x19)
        )
        if transformed == CHECK[pos]:
            candidate[pos] = current
            dfs(pos - 1)


dfs(len(CHECK) - 2)

for value in solutions:
    print(value)
    if value.startswith(b"moectf{") and value.endswith(b"}"):
        print("flag:", value.decode())
```

可打印字符范围内共有 8 个局部方程解，其中只有一个满足题目已知的 flag 格式：

```text
moectf{Art_i5_b14s7ing!!!!!}
```

如果没有 `moectf{...}` 这一格式约束，就应把每个候选送入程序中的 SHA-1 校验函数，而不是尝试逆向 SHA-1。哈希在这里是候选判定器，不是需要求逆的主要障碍。

## 方法总结

这类相邻字节变换应先寻找未被修改的边界字节，再沿依赖方向逐位回溯。只要单步关系不是一一映射，就必须保留全部分支，并在末端用格式、长度或哈希约束统一筛选。壳只影响静态分析入口；本题真正可复用的技巧是把非单射局部方程转成低分支搜索，并正确区分“生成候选”和“验证候选”。
