# SU_babyAI

## 题目简述
附件给出 `task.py` 和 `model.pth`。`task.py` 将 `FLAG` 转成字节向量，送入一个小型 PyTorch 卷积/线性网络，输出再添加 `[-160,160]` 小噪声并模 `q` 公开；`model.pth` 不是缺失材料，而是隐藏了权重。问题本质是带小噪声的模线性方程组，正文只需要保留模型结构、模数和噪声范围，不需要写入完整 `model.pth` 二进制内容。

## 解题过程
题目信息

附件里只有两个文件：

- task.py

- model.pth

题目提示是：

It seems like something is missing.

结合附件名和提示，第一反应是先审 task.py ，再看 model.pth 里到底存了什么。

代码审计

task.py 的核心逻辑并不复杂，本质上是把 FLAG 当作字节序列输入一个非常小的神经网络，然
后把输出加一点噪声后模 q 输出出来。

关键参数如下：

```
FLAG = b"SUCTF{fake_flag_xxx}"
q = 1000000007
n = len(FLAG) # 41
m = 15
```

模型结构：

```
self.conv = nn.Conv1d(1, 1, 3, stride=2, bias=False)
self.fc = nn.Linear(conv_out_size, m_out, bias=False)
```

然后权重会被随机初始化为 0 ~ q-1 之间的整数，并保存到 model.pth ：

```python
torch.save(model.state_dict(), "model.pth")
```

输出生成过程可以写成：

```
conv_out[i] = w0*x[2i] + w1*x[2i+1] + w2*x[2i+2]
Y[j] = sum(fc[j][i] * conv_out[i]) + noise (mod q)
```

其中 noise 满足：

```
noise ∈ [-160, 160]
```

题目给出的公开信息是：

```
n = 41
m = 15
q = 1000000007
Y = [776038603, 454677179, 277026269, 279042526, 78728856, 784454706,
29243312, 291698200, 137468500, 236943731, 733036662, 421311403, 340527174,
804823668, 379367062]
```

关键观察

1.`model.pth` 不是“缺失”了，而是权重就藏在里面

model.pth 是 PyTorch 的 state_dict ，虽然本地环境没有装 torch ，但它本质上是一个
zip 格式的归档文件，可以直接拆。

里面最重要的两个数据块是：

- model/data/0 ：conv.weight
- model/data/1 ：fc.weight

也就是说，题目里“似乎缺了点什么”，其实缺的不是文件，而是选手要主动意识到：

既然权重已经给了，那这个网络根本不是黑盒。

2. 整个网络本质上是一个模`q` 的线性方程组

因为没有 bias，也没有激活函数，所以整个模型是线性的。

设 flag 字节为：

```
x0, x1, ..., x40
```

卷积层 stride=2，kernel size=3，所以会得到 20 个卷积输出：

```
c_i = a0*x_{2i} + a1*x_{2i+1} + a2*x_{2i+2}
```

全连接层再做一次线性组合：

```
Y_j = Σ b_{j,i} * c_i + e_j (mod q)
```

把卷积展开后，就能整理成：

```
Y = A * X + E (mod q)
```

其中：

- `A` 是 `15 x 41` 的已知矩阵；
- `X` 是长度 41 的 flag 字节向量；
- `E` 是每一维都很小的噪声向量，满足 `|E_i| <= 160`。

这一步就是整个题的核心化简。

为什么 15 个方程还能解出 41 个字符

表面上看，15 < 41 ，方程数量远远不够。

但这里还有几个非常强的额外约束：

格式已知，以 SUCTF{ 开头、以 } 结尾• flag

字符基本都在可打印 ASCII 范围内• flag

- 噪声非常小，只有 ±160

- 模数 q = 1000000007 很大，远大于字符范围

这就把问题从“欠定线性方程组”变成了“带小误差的小范围整数解搜索”，本质上非常接近 LWE /
BDD / CVP 一类问题。

这种情况下，格方法是很自然的选择。

求解思路

1. 直接从`model.pth` 提取权重

因为 model.pth 是 zip，可以用 zipfile + struct 直接读出 float32：

```
with zipfile.ZipFile("model.pth") as archive:
conv = struct.unpack("<3f", archive.read("model/data/0"))
fc = struct.unpack("<300f", archive.read("model/data/1"))
```

再转成整数即可。

2. 展开得到总系数矩阵`A`

如果卷积核是 w_conv = [w0, w1, w2] ，全连接某一行是 w_fc[row][i] ，那么：

```
A[row][2i + 0] += w_fc[row][i] * w0
A[row][2i + 1] += w_fc[row][i] * w1
A[row][2i + 2] += w_fc[row][i] * w2
```

全部对 q 取模。

3. 先消掉已知字符

flag 头尾基本是板上钉钉的：

```
S U C T F { ... }
```

所以可以先把这些已知字符对应的贡献从 Y 中减掉，只留下未知位置。

4. 把字符平移到中心区间

可打印 ASCII 大概在 [32, 126] ，中心大约是 79 。

令未知字符：

```
x_i = 79 + u_i
```

那么 u_i 的范围就很小，大概落在 [-47, 47] 。

这样做的好处是，格里要找的向量会更短，更适合 LLL + Babai。

5. 构造格并做最近向量搜索

构造列基：

- 前 15 列是 q * e_i
- 后面每一列对应一个未知字符，列向量形如：

```
(A'_j, λ * e_j)
```

其中：

是未知字符在 15 个方程里的系数列• A'_j

是一个小权重，这里取 1 就够了• λ

目标向量取：

```
(Y' - A' * 79, 0, 0, ..., 0)
```

然后：

1. 对基做 LLL 约化

2. 用 Babai nearest plane 找最近格点

3. 还原出每个 u_i

4. 再加回 79 得到真实字符

本地脚本

```python
import struct
import zipfile

from sympy import Matrix

Q = 1_000_000_007
Y = [
776038603,
454677179,
277026269,
279042526,
78728856,
784454706,
29243312,
291698200,
137468500,
236943731,
733036662,
421311403,
340527174,
804823668,
379367062,

]
KNOWN = {
0: ord("S"),
1: ord("U"),
2: ord("C"),
3: ord("T"),
4: ord("F"),
5: ord("{"),
40: ord("}"),
}

def centered_mod(value):
if value > Q // 2:
value -= Q
return value

def gram_schmidt_columns(columns):
dim = len(columns)
length = len(columns[0])
ortho = [[0.0] * length for _ in range(dim)]
norms = [0.0] * dim

for i in range(dim):
vector = [float(x) for x in columns[i]]
for j in range(i):
if norms[j] == 0:
continue
mu = sum(vector[k] * ortho[j][k] for k in range(length)) / norms[j]
for k in range(length):
vector[k] -= mu * ortho[j][k]
ortho[i] = vector
norms[i] = sum(x * x for x in vector)
return ortho, norms

def babai_nearest_plane(columns, target):
ortho, norms = gram_schmidt_columns(columns)
coeffs = [0] * len(columns)
residue = [float(x) for x in target]

for i in range(len(columns) - 1, -1, -1):
if norms[i] == 0:
coeff = 0
else:
coeff = round(sum(residue[k] * ortho[i][k] for k in
range(len(target))) / norms[i])
coeffs[i] = int(coeff)
for k in range(len(target)):

residue[k] -= coeff * columns[i][k]
return coeffs

def load_weights(path):
with zipfile.ZipFile(path) as archive:
conv = list(map(int, struct.unpack("<3f",
archive.read("model/data/0"))))
fc = list(map(int, struct.unpack("<300f",
archive.read("model/data/1"))))
return conv, [fc[i * 20 : (i + 1) * 20] for i in range(15)]

def build_matrix(conv, fc):
matrix = [[0] * 41 for _ in range(15)]
for row in range(15):
for i in range(20):
for offset, weight in enumerate(conv):
matrix[row][2 * i + offset] = (matrix[row][2 * i + offset] +
fc[row][i] * weight) % Q
return matrix

def solve_flag(matrix):
unknown_positions = [index for index in range(41) if index not in KNOWN]
shifted_target = []

for row in range(15):
value = Y[row]
for index, known_value in KNOWN.items():
value = (value - matrix[row][index] * known_value) % Q
shifted_target.append(value)

midpoint = 79
unknown_count = len(unknown_positions)
target_top = []
for row in range(15):
value = shifted_target[row]
for position in unknown_positions:
value = (value - matrix[row][position] * midpoint) % Q
target_top.append(centered_mod(value))

dim = 15 + unknown_count
columns = []

for row in range(15):
column = [0] * dim
column[row] = Q
columns.append(column)

for offset, position in enumerate(unknown_positions):
column = [matrix[row][position] for row in range(15)] + [0] *
unknown_count
column[15 + offset] = 1
columns.append(column)

basis = Matrix(dim, dim, lambda r, c: columns[c][r])
reduced_rows, transform = basis.T.lll_transform()
reduced_columns = reduced_rows.T
reduced_basis = [[int(reduced_columns[r, c]) for r in range(dim)] for c in
range(dim)]

reduced_coeffs = babai_nearest_plane(reduced_basis, target_top + [0] *
unknown_count)
original_coeffs = list((transform.T *
Matrix(reduced_coeffs)).applyfunc(int))
solved_unknowns = [midpoint + value for value in original_coeffs[15:]]

flag_bytes = [KNOWN.get(index, 0) for index in range(41)]
for position, value in zip(unknown_positions, solved_unknowns):
flag_bytes[position] = value
return bytes(flag_bytes)

def verify(flag, matrix):
errors = []
for row in range(15):
value = sum(matrix[row][i] * flag[i] for i in range(41)) % Q
diff = (value - Y[row]) % Q
if diff > Q // 2:
diff -= Q
errors.append(diff)
return max(abs(error) for error in errors) <= 160, errors

def main():
conv, fc = load_weights("model.pth")
matrix = build_matrix(conv, fc)
flag = solve_flag(matrix)
ok, errors = verify(flag, matrix)
print(flag.decode())
print(errors)
if not ok:
raise SystemExit("verification failed")

if __name__ == "__main__":
main()
```

脚本会：

1. 从 model.pth 直接提取权重

2. 重建系数矩阵

3. 用 sympy 的 lll_transform() 做格约化

4. 用 Babai nearest plane 恢复未知字符

5. 最后校验所有噪声是否都在 [-160, 160] 范围内

验证结果

脚本跑出的结果为：

```
SUCTF{PyT0rch_m0del_c4n_h1d3_LWE_pr0bl3m}
```

对应残差为：

```
[-53, 105, 105, -55, 9, -17, 65, -2, 140, -111, 101, 76, 81, 126, -109]
```

可以看到每一项都满足：

```
|noise| <= 160
```

因此解是正确的。

最终 Flag

```
SUCTF{PyT0rch_m0del_c4n_h1d3_LWE_pr0bl3m}
```

## 方法总结
- 核心技巧：PyTorch 权重提取 + LWE/格约化
- 识别信号：公开输出满足模线性关系且噪声范围很小，附件包含模型权重。
- 复用要点：从 state_dict 提取权重，重建线性方程，扣除已知 flag 位置后用 LLL/Babai 找最近向量，再回代验证噪声界。
