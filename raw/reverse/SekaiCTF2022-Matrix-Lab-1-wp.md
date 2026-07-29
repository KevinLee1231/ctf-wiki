# Matrix Lab 1

## 题目简述

附件是一个 Java `.class` 文件。程序要求输入 43 个字符，并检查 `SEKAI{...}` 格式；中间 36 个字符会被排成 $6\times6$ 矩阵，顺时针旋转 $90^\circ$ 后按成对的外层行抽取、异或，最终与固定密文比较。

每一步都是可逆变换，因此不需要爆破，只需按相反顺序恢复矩阵。

## 解题过程

用 CFR、FernFlower 或 JADX 反编译 `Matrix_Lab_1.class`。静态字段的初始化为：

```java
Sekai.length = (int)Math.pow(2.0, 3.0) - 2;
```

所以 `length = 6`，flag 中间部分正好是 36 个字符。`solve()` 先将字符串按行写入矩阵，随后执行一轮原地环形交换。观察四个赋值方向可知，它把矩阵顺时针旋转了 $90^\circ$。

旋转后，程序依次抽取第 0/5、1/4、2/3 行：

```java
getArray(transform, 0, 5)
getArray(transform, 1, 4)
getArray(transform, 2, 3)
```

`getArray(m, a, b)` 的输出是第 `a` 行从左到右，再接第 `b` 行从右到左。`encrypt()` 又把这 12 个字符从中间向两侧交错取出：

```java
data[2 * k]     = array[5 - k];
data[2 * k + 1] = array[6 + k];
```

三个分块分别异或 2、1、0，拼接后必须等于：

```text
oz]{R]3l]]B#50es6O4tL23Etr3c10_F4TD2
```

逆向时，将密文每 12 字符切成一组，先撤销异或，再撤销交错读取；前 6 字符还原到上侧行，后 6 字符反转后还原到对应的下侧行。最后对整个矩阵逆时针旋转 $90^\circ$。

```python
cipher = "oz]{R]3l]]B#50es6O4tL23Etr3c10_F4TD2"
rows = [None] * 6

for layer, key in enumerate((2, 1, 0)):
    block = cipher[layer * 12:(layer + 1) * 12]
    mixed = [chr(ord(ch) ^ key) for ch in block]

    pair = [""] * 12
    for k in range(6):
        pair[5 - k] = mixed[2 * k]
        pair[6 + k] = mixed[2 * k + 1]

    rows[layer] = pair[:6]
    rows[5 - layer] = pair[6:][::-1]

# rows 是顺时针旋转后的矩阵；下面逆时针旋转一次。
original = [list(row) for row in zip(*rows)][::-1]
middle = "".join("".join(row) for row in original)
print(f"SEKAI{{{middle}}}")
```

输出为：

```text
SEKAI{m4tr1x_d3cryP710N_15_Fun_M4T3_@2D2D!}
```

## 方法总结

解题时应先把反编译代码拆成三个独立的可逆步骤：矩阵旋转、行对抽取、交错异或。最容易出错的是第二行的方向：`getArray()` 读取下侧行时已经反转，恢复时必须再反转一次。

遇到矩阵题不必凭视觉猜排列。为每次索引赋值写出明确映射，再用正向算法重新加密恢复结果，是检查旋转方向与行列顺序最可靠的方法。
