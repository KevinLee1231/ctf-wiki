# GoldDigger

## 题目简述

题目给出一个 64 位 ELF，程序要求命令行参数长度恰好为 28。校验逻辑把输入按一张索引置换表重新取值，每个字符加 4 后与目标数组比较。失败提示中的二进制字符串解码为 `Try Harder!!!`，只是提示而不是解法关键。

目标字符串本身包含 `TamilCTF{...}`，因此还原出的 28 字符就是完整 flag。

## 解题过程

### 关键观察

主函数逻辑：

```c
if (argc < 2) fail;
if (strlen(argv[1]) != 28) fail;
if (validate(argv[1]) == 0) success;
```

`validate` 先构造索引表，再比较：

```c
index[i] = DAT_00102020[i] ^ 0x12;
result[i] = input[index[i]] + 4;
result[i] == DAT_001020a0[i]
```

因此逆向公式是：

```text
input[index[i]] = target[i] - 4
```

### 求解步骤

从 `.rodata` 提取：

```python
index_raw = [22,10,31,16,24,27,9,11,30,3,8,20,29,0,7,17,
             23,19,21,28,18,2,6,4,25,5,26,1]
target = [112,83,106,113,52,125,129,80,99,72,104,88,89,99,
          73,109,71,101,74,115,88,114,76,99,121,75,127,120]

flag = [""] * 28
for i, v in enumerate(index_raw):
    flag[v ^ 0x12] = chr(target[i] - 4)
print("".join(flag))
```

输出：

```text
TamilCTF{y0u_foUnD_tHE_GOLd}
```

## 方法总结

- 核心技巧：数组置换 + 常量偏移的逆向还原。
- 识别信号：验证函数中出现 `input[index[i]] + const == target[i]`，通常可以直接按目标数组反填输入。
- 复用要点：先确认索引表是否被运行时变换；本题 `^ 0x12` 是索引表的一部分，漏掉会导致 flag 顺序错乱。
