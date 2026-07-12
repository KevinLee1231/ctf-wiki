# SUCTF2026-我不是神偷

## 题目简述
本题是上一题的加强版：附件 `app.py` 保留了 Flask 服务接口和旧结构线索，但与线上 `10001` 的真实权重形状不一致，附件中的 `8x8/7x7` 结构只是历史/烟雾信息。题面提示“先看行为，再看命名”，所以解法先用线上报错确定当前服务实际包含两层 `4x4` 卷积和共享线性层，再借历史服务恢复共享的 `linear.weight/linear.bias`，最后把当前两层卷积合成的 `7x7` 等效核分解回两层 `4x4`。

## 解题过程
具体来说：

1. 用 /flag 的报错先摸清线上真实模型形状。

2. 用 /predict 的线性性质，把整个模型恢复成一个仿射映射。

3. 利用旁边的历史服务 10002 先恢复出共享的 linear.weight / linear.bias 。

4. 再回到 10001 ，把当前两层 4x4 卷积合成后的 7x7 等效核提出来。

5. 对这个 7x7 核做 4x4 + 4x4 因式分解，再结合题面给的两个 bias 线索试层顺序。

6. 唯一能过 /flag 的组合就是正确答案。

1. 附件`app.py` 不是线上真实快照

附件里写的是：

```
self.conv = nn.Conv2d(1, 1, (8, 8), stride=1)
self.conv1 = nn.Conv2d(1, 1, (7, 7), stride=1)
```

但线上 10001 的 /flag 实际会告诉我们：

期待的是 4x4 • conv.weight

期待的也是 4x4 • conv1.weight

所以附件只能当热身材料，不能当真相。

2. 命名会误导你

题面已经明说了：先看行为，再看命名。

这句话非常关键，因为：

- 附件命名和线上结构不一致

这个词也未必对应当前 conv / conv1 的命名• legacy
- 最终能过校验的，是“行为一致”的模型，不是“名字看起来像”的模型

3.`/predict` 里有一个 `view(-1)` 小坑

线上 forward 本质是先卷积，再直接 view(-1) 喂给线性层。

所以虽然正常输入是单张图，但实际上只要总元素数能凑成 256，也会被吃进去，比如：
- 1 x 1 x 22 x 22
- 16 x 1 x 10 x 10
- 64 x 1 x 8 x 8
- 256 x 1 x 7 x 7

不过这题最后并不需要依赖这个坑，直接用普通 basis query 就能做完。

### 第一步：先确认线上真实结构

10001 当前服务

对 /flag 提交伪造 state dict，可以直接拿到形状信息：
- linear.weight : 256 x 256
- linear.bias : 256
- conv.weight : 1 x 1 x 4 x 4
- conv.bias : 1
- conv1.weight : 1 x 1 x 4 x 4
- conv1.bias : 1

再对 /predict 做输入尺寸测试，可以发现单图合法输入是 22 x 22 。

因此当前线上真实主路径是：

```
Input(22x22)
-> Conv(4x4)
-> Conv(4x4)
-> 16x16
-> Flatten(256)
-> Linear(256->256)

历史服务
```

继续探测会发现 10002 也是同类服务，但它只有一层卷积：

- 输入是 19 x 19

是 4 x 4 • conv.weight
- conv1.* 是多余键

所以 10002 的结构是：

```
Input(19x19)
-> Conv(4x4)
-> 16x16
-> Flatten(256)
-> Linear(256->256)
```

这恰好和题面“小S保留了线性层与一层卷积层不变”对上了：

10002 很像“旧版本”，可以拿来恢复被保留的线性层。

### 第二步：把`/predict` 恢复成仿射映射

因为整个网络没有激活函数，所以它对输入其实是一个标准仿射变换：

```
y = Mx + b
```

其中：

- x 是拉平后的输入
- M 是输出对输入的线性映射矩阵
- b 是全零输入时的输出

恢复方法很直接：

1. 查询一次全零输入，得到 b

2. 对每个像素位置打一个 basis e_i

3. 计算 f(e_i) - b ，这就是矩阵 M 的第 i 列

### 查询次数

输入是 22x22 ，所以要 1 + 484 = 485 次• 10001

输入是 19x19 ，所以要 1 + 361 = 362 次• 10002

脚本里就是这么做的，缓存目录分别是：

- cache/
- cache10002/

### 第三步：先打通 10002，恢复共享线性层

### 3.1 10002 的卷积核可以单独恢复

对 10002 而言，结构是：

```
Input
-> Conv(4x4, bias = ?)
-> 16x16 hidden
-> Linear(256->256)
```

因为只有一层卷积，所以每个 hidden 单元都对应“同一个 4x4 核的平移版本”。

我们可以在 `M2` 的行空间里找一个**只落在某个 `4x4` 窗口内**的向量，这样就能直接把真实卷积
核抠出来。

最终恢复出的 10002 卷积是：

```
[[-6, -10, 1, -4],
[ 6, -1, 8, 8],
[ 9, -7, 6, -4],
[-5, 6, 8, -6]]
```

对应 bias 为：

```
4
```

### 3.2 用 10002 解出`linear.weight / linear.bias`

把上面的卷积核记作 G 。

对于 16x16 的每个 hidden 位置，它在输入上的作用就是一个平移后的 G 。

把这 256 个平移版按行堆起来，得到 hidden 映射矩阵 H2 。

则有：

```
M2 = W * H2
b2 = W * (4 * 1_256) + D
```

其中：

- W 是线性层权重

是线性层 bias• D

所以：

```
W = M2 * H2^T * (H2 * H2^T)^(-1)
D = b2 - W * (4 * 1_256)
```

实测可以发现恢复出的 W 几乎是严格整数矩阵，直接 round 就能过校验。

这一步非常关键，因为它证明：

- 10002 确实是可精确恢复的旧版本
- linear.weight / linear.bias 可以被当成稳定锚点

### 第四步：回到 10001，剥离线性层

既然题面说保留了线性层，而我们又已经从 10002 精确恢复了这层，那么对 10001 有：

```
M1 = W * H1
b1 = W * c + D
```

于是：

```
H1 = W^(-1) * M1
c = W^(-1) * (b1 - D)
```

这里：

- H1 是“当前两层卷积合起来”对输入的 hidden 映射

是卷积部分合成后的等效 bias• c

实测 c 的 256 个分量几乎是常数，均值约：

```
12.6126164
```

这进一步说明：

- 线性层确实是复用的

- 前面剥离线性层的思路是对的

### 第五步：提取当前等效`7x7` 卷积核

当前服务有两层 4x4 卷积，串起来以后，对输入的等效作用范围就是 7x7 。

对 H1 的第 k 行，把它 reshape 回 22x22 ，再取对应 hidden 位置上的 7x7 patch，所有位置
平均后就能得到当前服务的等效卷积核 K 。

得到的 K 大致如下：

```
[[ -8.395328, -0.146497, -11.435442, -13.526918, -52.755468, -17.323286,
9.776742],
[ -9.911057, -15.831160, -50.778461, 20.806315, 21.143627, 37.533756,
-21.546585],
[-46.598348, -72.955344, -67.868516, -80.275761, -16.089422, -46.519452,
25.694825],
[-25.968793, -15.353923, 7.311627, 75.106459, 175.935488, -86.333268,
-1.332327],
[-43.252760, -66.199486, 53.472684, 23.355333, -18.605255, 121.368775,
-32.058099],
[-20.859178, -22.974305, 66.145563, 58.704583, -43.059285, -13.727155,
25.636175],

[ -0.951334, 5.359760, 29.645660, 40.666994, 11.434185, -11.399581,
-5.454383]]
```

这不是最终要提交的参数，但它是“真实两层卷积合成后的结果”。

### 第六步：把`7x7` 等效核分解回两层 `4x4`

### 6.1 分解问题

如果第一层卷积核是 A ，第二层卷积核是 B ，那么它们合成后的等效核满足：

```
K = full(B, A)
```

这里 full 表示两层相关运算叠加后的 7x7 结果。

我这里直接用 LBFGS 对两个 4x4 矩阵做数值优化，让：

```
full(B, A) ≈ K
```

得到一组高精度分解。

### 6.2 分解存在缩放不唯一性

如果 (A, B) 可以组成 K ，那么：

```
(sA, B/s)
```

也会组成同一个 K 。

所以单靠 K 本身，只能恢复到一个“缩放族”，还差最后一个约束。

### 第七步：用题面 bias 线索锁定真实层顺序

题面给了两条非常关键的线索：

```
legacy -5.640393257141113
conv1.bias = -4.398319721221924
```

但题面也说了命名不可信，所以这里不能直接认定：

一定是当前 conv.bias • legacy

一定还是当前意义上的 conv1.bias • conv1.bias

必须把两种层顺序、两种 bias 对应关系都试掉。

### 7.1 等效 bias 公式

设：

- 第一层 bias 为 b0

- 第二层 bias 为 b1

- 第二层卷积核为 B

那么两层合成后的 hidden 等效 bias 为：

```
c = b1 + b0 * sum(B)
```

而我们前面已经从 10001 直接恢复出了 c 的均值。

又因为分解后的核有缩放不唯一性，若当前取到的是基础分解 (first, second) ，并令：

```
conv = s * first
conv1 = second / s
```

则有：

```
effective_bias = conv1_bias + conv_bias * sum(second / s)
scale = conv_bias * sum(second) / (effective_bias - conv1_bias)
```

这就把最后一个自由度也锁死了。

### 7.2 实际尝试结果

把：

- 两种层顺序

- 两种 bias 对应方式

都枚举掉以后，只有下面这一组能过：

- 正确层顺序是：交换后的顺序
- conv.bias = -5.640393257141113
- conv1.bias = -4.398319721221924

### 一组成功过校验的参数

下面是一组实际通过 10001 /flag 校验的卷积参数。

`conv.weight`

```
[[ 5.8819090, 8.2369980, 9.2516950, -3.4896755],
[ 1.9883984, -0.03455263, 8.0709010, 0.2524918],
[ 6.5513415, 5.3240304, -6.5983615, 2.9656336],
[ 3.1902468, 3.4249732, -0.80356663, -0.9793773]]
```

`conv.bias`

```
-5.640393257141113
```

`conv1.weight`

```
[[-1.4273129, 1.9738903, -2.4633690, -2.8016138],
[-1.2025006, -1.6831887, -1.5815915, 5.9716706],
[-5.9260454, -4.4491963, 5.5439115, -9.3119190],
[-0.29820102, 2.0001895, 7.0701294, 5.5692230]]
```

`conv1.bias`

```
-4.398319721221924
```

说明：

- 这是一组真实过了 /flag 的参数

- 线性层权重过大，这里不贴全矩阵

- 完整模型文件已经由脚本保存为 recovered_model_10001.pth

### 自动化脚本说明

```python
import base64
import io
import time
from pathlib import Path

import numpy as np
import requests
import torch

CURRENT_URL = "http://<target>:10001"
WARMUP_URL = "http://<target>:10002"

CURRENT_INPUT = 22
WARMUP_INPUT = 19
HIDDEN_SIZE = 16

CACHE_CURRENT = Path("cache")
CACHE_WARMUP = Path("cache10002")

# 10002 已经能精确验证通过，因此把它当成共享线性层的锚点。
WARMUP_CONV = np.array(
[
[-6, -10, 1, -4],
[6, -1, 8, 8],
[9, -7, 6, -4],
[-5, 6, 8, -6],
],
dtype=np.float64,
)
WARMUP_CONV_BIAS = 4.0

# 题面里给出的两条偏置线索。
LEGACY_BIAS = -5.640393257141113
CONV1_BIAS_CLUE = -4.398319721221924

def query_predict(url: str, image: np.ndarray) -> np.ndarray:
payload = {"image": image.tolist()}
last_error = None

for attempt in range(8):
try:
response = requests.post(f"{url}/predict", json=payload,
timeout=20)
response.raise_for_status()
return np.array(response.json()["prediction"], dtype=np.float64)
except Exception as exc:
last_error = exc
time.sleep(min(2.0 * (attempt + 1), 10.0))
raise RuntimeError(f"predict failed for {url}: {last_error}")

def recover_affine(url: str, input_size: int, cache_dir: Path, width: int) ->
tuple[np.ndarray, np.ndarray]:
cache_dir.mkdir(exist_ok=True)
zero = np.zeros((1, 1, input_size, input_size), dtype=np.float32)

bias_path = cache_dir / "bias.npy"
if bias_path.exists():
bias = np.load(bias_path)
else:
bias = query_predict(url, zero)
np.save(bias_path, bias)

cols = []
for idx in range(input_size * input_size):
col_path = cache_dir / f"col_{idx:0{width}d}.npy"
if col_path.exists():
col = np.load(col_path)
else:
image = zero.copy()
r, c = divmod(idx, input_size)
image[0, 0, r, c] = 1.0
col = query_predict(url, image) - bias
np.save(col_path, col)
cols.append(col)

return np.stack(cols, axis=1), bias

def build_warmup_hidden(kernel: np.ndarray) -> np.ndarray:
rows = []
for top in range(HIDDEN_SIZE):
for left in range(HIDDEN_SIZE):
canvas = np.zeros((WARMUP_INPUT, WARMUP_INPUT), dtype=np.float64)
canvas[top : top + 4, left : left + 4] = kernel
rows.append(canvas.reshape(-1))
return np.stack(rows, axis=0)

def recover_shared_linear() -> tuple[np.ndarray, np.ndarray]:
matrix, bias = recover_affine(WARMUP_URL, WARMUP_INPUT, CACHE_WARMUP, 3)
hidden = build_warmup_hidden(WARMUP_CONV)
gram = hidden @ hidden.T
weight = matrix @ hidden.T @ np.linalg.inv(gram)

# 10002 上它基本是严格整数矩阵，round 之后可直接过 /flag。
weight = np.round(weight)
linear_bias = bias - weight @ np.full(256, WARMUP_CONV_BIAS,
dtype=np.float64)
return weight, linear_bias

def recover_current_effective_kernel(weight: np.ndarray, linear_bias:
np.ndarray) -> tuple[np.ndarray, float]:
matrix, bias = recover_affine(CURRENT_URL, CURRENT_INPUT, CACHE_CURRENT, 3)
hidden = np.linalg.inv(weight) @ matrix
bias_vec = np.linalg.inv(weight) @ (bias - linear_bias)

patches = []
for idx in range(256):
top, left = divmod(idx, HIDDEN_SIZE)
patch = hidden[idx].reshape(CURRENT_INPUT, CURRENT_INPUT)[top : top +
7, left : left + 7]
patches.append(patch)

kernel = np.mean(np.stack(patches, axis=0), axis=0)
return kernel, float(np.mean(bias_vec))

def compose_full(second: torch.Tensor, first: torch.Tensor) -> torch.Tensor:
out = torch.zeros((7, 7), dtype=torch.float64)
for i in range(4):
for j in range(4):
out[i : i + 4, j : j + 4] += second[i, j] * first
return out

def factorize_effective_kernel(kernel: np.ndarray, seeds: int = 24) ->
tuple[np.ndarray, np.ndarray, float]:
target = torch.tensor(kernel, dtype=torch.float64)
best_first = None
best_second = None
best_error = None

for seed in range(seeds):
torch.manual_seed(seed)
first = torch.randn(4, 4, dtype=torch.float64, requires_grad=True)
second = torch.randn(4, 4, dtype=torch.float64, requires_grad=True)
optimizer = torch.optim.LBFGS(

[first, second],
lr=0.5,
max_iter=300,
line_search_fn="strong_wolfe",
)

def closure() -> torch.Tensor:
optimizer.zero_grad()
loss = ((compose_full(second, first) - target) ** 2).mean()
loss.backward()
return loss

optimizer.step(closure)

with torch.no_grad():
error = torch.max(torch.abs(compose_full(second, first) -
target)).item()
if best_error is None or error < best_error:
best_error = error
best_first = first.detach().clone()
best_second = second.detach().clone()

if best_first is None or best_second is None or best_error is None:
raise RuntimeError("failed to factorize effective kernel")

# 只做数值重平衡，不改变组合后的 7x7 等效核。
first_norm = best_first.norm().item()
second_norm = best_second.norm().item()
if first_norm > 0 and second_norm > 0:
scale = (second_norm / first_norm) ** 0.5
best_first *= scale
best_second /= scale

return best_first.numpy(), best_second.numpy(), best_error

def build_state_dict(
weight: np.ndarray,
linear_bias: np.ndarray,
conv: np.ndarray,
conv_bias: float,
conv1: np.ndarray,
conv1_bias: float,
) -> dict[str, torch.Tensor]:
return {
"linear.weight": torch.tensor(weight, dtype=torch.float32),
"linear.bias": torch.tensor(linear_bias, dtype=torch.float32),

"conv.weight": torch.tensor(conv.reshape(1, 1, 4, 4),
dtype=torch.float32),
"conv.bias": torch.tensor([conv_bias], dtype=torch.float32),
"conv1.weight": torch.tensor(conv1.reshape(1, 1, 4, 4),
dtype=torch.float32),
"conv1.bias": torch.tensor([conv1_bias], dtype=torch.float32),
}

def submit_model(state_dict: dict[str, torch.Tensor]) -> requests.Response:
buffer = io.BytesIO()
torch.save(state_dict, buffer)
payload = {"model": base64.b64encode(buffer.getvalue()).decode()}
return requests.post(f"{CURRENT_URL}/flag", json=payload, timeout=20)

def solve_current_model() -> tuple[str, dict[str, torch.Tensor]]:
weight, linear_bias = recover_shared_linear()
kernel, effective_bias = recover_current_effective_kernel(weight,
linear_bias)
first_base, second_base, factor_error = factorize_effective_kernel(kernel)

print(f"shared linear recovered, factorization max error =
{factor_error:.6g}")

bias_pairs = [
(LEGACY_BIAS, CONV1_BIAS_CLUE),
(CONV1_BIAS_CLUE, LEGACY_BIAS),
]
orders = [
("base-order", first_base, second_base),
("swapped-order", second_base, first_base),
]

for label, first, second in orders:
for conv_bias, conv1_bias in bias_pairs:
scale = conv_bias * np.sum(second) / (effective_bias - conv1_bias)
conv = scale * first
conv1 = second / scale
state_dict = build_state_dict(weight, linear_bias, conv,
conv_bias, conv1, conv1_bias)
response = submit_model(state_dict)
print(label, conv_bias, conv1_bias, response.status_code,
response.text[:120])
if response.ok and "flag" in response.text:
return response.text, state_dict

raise RuntimeError("no candidate passed /flag")

def main() -> None:
response_text, state_dict = solve_current_model()
print(response_text)
output_path = Path("recovered_model_10001.pth")
torch.save(state_dict, output_path)
print(f"saved recovered model to {output_path}")

if __name__ == "__main__":
main()
```

脚本会自动完成：

1. 从 10002 恢复共享线性层

2. 从 10001 恢复等效 7x7 核

3. 做两层 4x4 因式分解

4. 结合 bias 线索试层顺序

5. 提交 /flag

6. 保存恢复出的模型到 recovered_model_10001.pth

实测输出中会出现：

```
shared linear recovered, factorization max error = ...
...
200 {"flag":"Here is your flag: ...
SUCTF{v3r1fy_b3h4v10r_n0t_h1st0ry_7a4c9d21}"}
```

## 方法总结
- 核心技巧：多服务行为对比与线性模型分解
- 识别信号：同一题组存在历史服务和当前服务，且 `/flag` 报错暴露真实 state_dict 形状。
- 复用要点：先从旧服务恢复共享层作为锚点，再从当前服务剥离线性层，把两层卷积视为一个等效核并做因式分解。
