# Gaming VM

## 题目简述

这是一道自定义 VM（Quake3 风格字节码）逆向题，核心逻辑在 `challenge/cake.c` 中仍保留了主判定语义。

关键机制是：

1. 读取用户输入并要求长度 `42`。
2. 每个字符先异或 `0x07`。
3. 基于 `seed = 1` 的 LCG（`seed = seed * 0x41c64e6d + 0x6073`）生成伪随机索引，对字符串做一轮从尾到头的 Fisher-Yates 风格交换。
4. 与固定字符串 `DDt4zNXXuAjk476XpsNs6bluNwfJVlQbXaSi|XfrXF` 比较。

源码里的交换逻辑可直接读到（节选）：

```c
for (i = n - 1; i > 0; i--) {
    j = rand() % i;
    c = flag[j];
    flag[j] = flag[i];
    flag[i] = c;
}

if (strcmp(flag, "DDt4zNXXuAjk476XpsNs6bluNwfJVlQbXaSi|XfrXF") != 0)
    goto wrong;
```

官方 `gen.py` 已给出目标字符串（`flag` 变量），说明这是典型“已知混淆过程逆推”题。

## 解题过程

### 关键观察

`rand()` 与 `swap` 仅与长度、`seed=1`、和常量 LCG 有关，完全可逆。只要把加密/混淆交换序列按逆序回放，再做异或还原，就能直接得到明文。

### 求解步骤

1. 复现前向 shuffle 过程，记录每一步 `(i, j)`，其中 `i` 从 `41` 递减到 `1`。
2. 从加密字符串开始，按记录的 swaps 反向执行（`i` 从小到大对应逆序）进行去混淆。
3. 对每个字符再异或一次 `0x07`，得到明文。

复现代码如下：

```python
enc = "DDt4zNXXuAjk476XpsNs6bluNwfJVlQbXaSi|XfrXF"
seed = 1
n = len(enc)

ops = []
for i in range(n - 1, 0, -1):
    seed = (seed * 0x41c64e6d + 0x6073) & 0x7fffffff
    ops.append((i, seed % i))

arr = list(enc)
for i, j in reversed(ops):
    arr[i], arr[j] = arr[j], arr[i]

flag = ''.join(chr(ord(c) ^ 7) for c in arr)
print(flag)
```

输出为：

```text
CakeCTF{A_s1mpl3_VM_wr1tt3n_f0r_Quake_III}
```

### 验证

将该结果再按原逻辑重新 XOR+shuffle 一次，得到的就是源码中的目标串，且长度与类型约束一致，因此逻辑闭环成立。

## 方法总结

- 核心技巧：把固定种子伪随机洗牌转成 swap 序列，再逆序还原。
- 识别信号：`rand` 洗牌、固定长度、固定对照串，且没有外部随机输入时，优先走纯函数逆向。
- 复用要点：每次出现“先加密再比对”的题，优先确认是否有可逆置换；先写出逆向 swap 再验证重放。
