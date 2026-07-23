# UMDCTF 2025 - spindle

## 题目简述

题目给出一个 $5\times5$ 字母网格。每一步选择同一行或同一列上的两个端点：
按选择方向读出的字符串必须在 `dictionary.txt` 中，随后该线段会被反转。目标是在
不超过 8 步内，让任意完整行或列变成 `FANUM`。

这是一道有限状态的单词与路径搜索题，不应因网页界面归入 Web，也没有更合适的正式
安全方向，因此放入 `_unclassified`。

## 解题过程

初始网格是：

```text
FNEHA
AKELB
VMTKE
OGUYU
PJPFN
```

`app.py` 中的 `get_word` 表明，端点顺序会决定校验单词的读取方向；通过校验后，
对应区间才会反转。可以把整个 25 字符网格作为状态，枚举每行、每列的所有端点对，
只保留正向或反向字符串在词典中的动作，再用广度优先搜索限制深度为 8。

下面给出一条实际可验证的 8 步路径。坐标均为从 0 开始的
`(row, column)`；“合法单词”按提交端点的方向读取。

1. `(1,1) -> (1,0)`，单词 `KA`：

   ```text
   FNEHA
   KAELB
   VMTKE
   OGUYU
   PJPFN
   ```

2. `(3,2) -> (2,2)`，单词 `UT`：

   ```text
   FNEHA
   KAELB
   VMUKE
   OGTYU
   PJPFN
   ```

3. `(0,1) -> (2,1)`，单词 `NAM`：

   ```text
   FMEHA
   KAELB
   VNUKE
   OGTYU
   PJPFN
   ```

4. `(2,1) -> (2,4)`，单词 `NUKE`：

   ```text
   FMEHA
   KAELB
   VEKUN
   OGTYU
   PJPFN
   ```

5. `(0,4) -> (0,1)`，单词 `AHEM`：

   ```text
   FAHEM
   KAELB
   VEKUN
   OGTYU
   PJPFN
   ```

6. `(2,3) -> (0,3)`，单词 `ULE`：

   ```text
   FAHUM
   KAELB
   VEKEN
   OGTYU
   PJPFN
   ```

7. `(2,2) -> (2,4)`，单词 `KEN`：

   ```text
   FAHUM
   KAELB
   VENEK
   OGTYU
   PJPFN
   ```

8. `(0,2) -> (2,2)`，单词 `HEN`：

   ```text
   FANUM
   KAELB
   VEHEK
   OGTYU
   PJPFN
   ```

第一行已成为 `FANUM`，且步数恰好为 8，服务返回：

```text
UMDCTF{spindle_is_the_ultimate_crossover_between_shape_rotators_and_wordcels}
```

## 方法总结

本题应先从源码精确恢复“端点方向、词典校验、区间反转”的状态转移，再做有深度上限的
搜索。网格只有 25 个字符，动作又被词典强约束，双向广度优先搜索或带去重的 BFS
都能有效缩小空间；最终还应逐步回放，避免只给出一个无法在界面复现的终局。
