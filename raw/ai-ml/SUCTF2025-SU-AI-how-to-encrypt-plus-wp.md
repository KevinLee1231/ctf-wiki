# SU_AI_how_to_encrypt_plus

## 题目简述

题目把 flag 的每个字符编码成 9 位二进制，再依次送入一个没有激活函数的三层 PyTorch 网络：

```text
9n 位输入
  -> 3×3、stride=3 的卷积
  -> n 到 n² 的线性层
  -> 2×2、stride=1、padding=1 的卷积
  -> (n+1)×(n+1) 密文
```

附件包含 `model.pth`、`ciphertext.txt` 和加密脚本。密文是 $49\times49$，故 $n=48$。网络完全由线性/仿射运算组成，而且卷积权重经过特殊设计，三层都可以从输出反推输入。

仓库内有官方 `wp.py`。当前指定 venv 与 WSL `ctf-tools` 都没有可用的 PyTorch，且没有擅自安装依赖，因此本文没有宣称本机重新加载模型成功；层逆向、关键参数和结果同时由本地官方脚本及[公开独立解题 notebook](https://github.com/OutWrest/blog-handouts/blob/main/suctf2025-ai-writeups/SU_AI_how_to_encrypt_plus/solve.ipynb)交叉核对。

## 解题过程

### 1. 逆最后一层卷积

最后一层参数为：

```text
weight = [[ 7, -5],
          [ 9, -7]]
bias   = -7
```

输入是 $n\times n$，经 `padding=1` 后输出 $(n+1)\times(n+1)$。对左上到右下的扫描顺序，输出满足

$$
y_{i,j}
=w_{00}x_{i-1,j-1}
+w_{01}x_{i-1,j}
+w_{10}x_{i,j-1}
+w_{11}x_{i,j}
+b,
$$

越界的 $x$ 视为零。由于 $w_{11}=-7\ne0$，已恢复的上方和左方元素足以递推当前元素：

$$
x_{i,j}
=\frac{
y_{i,j}-b
-w_{00}x_{i-1,j-1}
-w_{01}x_{i-1,j}
-w_{10}x_{i,j-1}
}{w_{11}}.
$$

实现时只需使用输出左上角的 $48\times48$ 区域：

```python
x = np.zeros((n, n), dtype=np.float64)

for i in range(n):
    for j in range(n):
        value = ciphertext[i, j] - conv_bias
        if i > 0 and j > 0:
            value -= w00 * x[i - 1, j - 1]
        if i > 0:
            value -= w01 * x[i - 1, j]
        if j > 0:
            value -= w10 * x[i, j - 1]
        x[i, j] = value / w11
```

### 2. 逆线性层

线性层执行

$$
y=Wx+b,
$$

其中 $W$ 的形状为 $2304\times48$。它不是方阵，不能直接求普通逆；正确做法是使用 Moore–Penrose 伪逆：

$$
x=W^{+}(y-b).
$$

```python
linear_output = x.reshape(n * n) - linear_bias
packed_values = np.linalg.pinv(linear_weight) @ linear_output
```

权重是整数且矩阵满列秩，因此数值误差只需在下一步用 `round()` 消除。

### 3. 逆第一层卷积并恢复字符

第一层卷积核固定为：

```text
[[  1,   2,   4],
 [  8,  16,  32],
 [ 64, 128, 256]]
```

输入每个位置都是 `0` 或 `1`，每个互不重叠的 $3\times3$ 块因此被编码为一个 9 位整数。减去该层 bias 后，连续取最低位就能按原位置恢复九个 bit：

```python
bits = []
conv1_bias = round(float(model["conv1.bias"][0]))

for value in packed_values:
    value = round(float(value)) - conv1_bias
    for _ in range(9):
        bits.append(value & 1)
        value >>= 1
```

加密脚本使用 `format(ord(char), "09b")` 生成每个字符的九位串，因此再每九位按大端二进制转回字符：

```python
flag = "".join(
    chr(int("".join(map(str, bits[i:i + 9])), 2))
    for i in range(0, len(bits), 9)
)
print(flag)
```

恢复结果为：

```text
SUCTF{Mi_sika_mosi!Mi_muhe_mita,mita_movo_lata!}
```

## 方法总结

“使用神经网络”不等于“不可逆”。本题没有 ReLU、池化、取整或随机采样，整个网络只是已知参数的仿射变换。最后一层卷积可利用边界和非零对角系数递推，线性层用伪逆恢复，第一层则把二进制位做了无碰撞的加权打包。分析此类模型加密题时，应先逐层检查信息是否真的丢失，再决定使用代数逆、约束求解还是优化攻击；直接对 432 个输入变量做梯度搜索既慢，也忽略了题目刻意保留的精确结构。
