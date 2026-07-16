# 旧吊带袜天使：想吃真蛋糕的Stocking

## 题目简述

题目允许上传 `.pth` 模型参数，并用上传后的 `SimpleDessertClassifier` 判断图片属于 `Cake`、`Porisoned_Apple` 还是 `Other`。当蛋糕置信度小于 24%，且毒苹果置信度高于蛋糕置信度时，接口会在 JSON 中返回 flag。

服务端使用 `torch.load(..., weights_only=True)` 读取 `state_dict`，再以默认的严格模式加载到固定网络结构中。因此本题不是通过 pickle 执行任意代码，而是构造键名和形状完全合法、输出层参数被恶意修改的模型投毒文件，使任何输入都稳定预测为毒苹果。

## 解题过程

### 定位 flag 条件与输出层

`/test_cake` 对模型输出做 softmax，并检查：

```python
if cake_confidence < 24 and poisoned_apple_confidence > cake_confidence:
    result['flag'] = os.environ.get('FLAG')
    result['message'] = 'Warning Cake'
```

分类器最后一层是 `nn.Linear(128, 3)`，三个输出依次对应蛋糕、毒苹果和其他：

```python
self.classifier = nn.Sequential(
    nn.Linear(128 * 7 * 7, 256), nn.ReLU(), nn.Dropout(0.3),
    nn.Linear(256, 128), nn.ReLU(),
    nn.Linear(128, 3),
)
```

其参数在 `state_dict` 中的键名和形状为：

```text
classifier.5.weight  -> (3, 128)
classifier.5.bias    -> (3,)
```

若把输出层权重全部置零，网络输出就与图片和前面所有层无关，只由三个偏置决定。令偏置为 `[-10, 10, 0]` 后，softmax 中毒苹果类别接近 100%，蛋糕类别接近 0%，必然满足 flag 条件。

### 生成合法的投毒参数文件

加载接口的安全限制为：

```python
state_dict = torch.load(model_path, map_location='cpu', weights_only=True)
new_model = SimpleDessertClassifier()
new_model.load_state_dict(state_dict)
new_model.eval()
```

`load_state_dict()` 默认 `strict=True`，所以不能只上传最后一层，也不能改变张量形状。最稳妥的做法是从题目附件导入同一个模型类，保留完整参数字典，只修改输出层：

```python
import torch
from model_server import SimpleDessertClassifier

model = SimpleDessertClassifier()
state_dict = model.state_dict()

weight_key = "classifier.5.weight"
bias_key = "classifier.5.bias"

state_dict[weight_key] = torch.zeros_like(state_dict[weight_key])
state_dict[bias_key] = torch.tensor(
    [-10.0, 10.0, 0.0],
    dtype=state_dict[bias_key].dtype,
)

torch.save(state_dict, "poisoned_fixed.pth")
```

生成文件包含网络需要的全部键和正确形状，同时 `weights_only=True` 可以正常读取其中的张量。

### 上传模型并触发判定

先上传投毒后的参数文件：

```bash
curl -F "model=@poisoned_fixed.pth;filename=poisoned_fixed.pth" \
  http://target/upload_model
```

返回 `success: true` 后，向测试接口上传任意能被 Pillow 正常解析的图片：

```bash
curl -F "image=@cake.png" \
  http://target/test_cake
```

由于输出层权重为零，换任何图片都会得到几乎相同的结果。响应中出现：

```json
{
  "cake_confidence": 0.0,
  "poisoned_apple_confidence": 100.0,
  "predicted_class": "Porisoned_Apple",
  "flag": "0xGame{AI_Stocking_Bakery_Poisoned_Cake}",
  "message": "Warning Cake"
}
```

最终 flag：

```text
0xGame{AI_Stocking_Bakery_Poisoned_Cake}
```

## 方法总结

本题考查模型参数完整性，而不是不安全反序列化。`weights_only=True` 限制了可反序列化对象，却不会判断张量参数是否可信；只要攻击者能上传结构合法的 `state_dict`，就能通过控制最后一层权重和偏置决定所有分类结果。生产环境需要对模型制品做签名、来源验证和行为测试，不能把“能成功加载”当作“模型可信”。
