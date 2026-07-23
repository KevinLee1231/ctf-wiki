# the worm strikes back

## 题目简述

本题与 `attack of the worm` 使用同一个 ResNet-18 二分类器，但不再允许修改固定图片的少量像素。我们必须提交一个 40×40 RGB 图像补丁；服务会对补丁随机旋转 $0^\circ$、$90^\circ$、$180^\circ$ 或 $270^\circ$，再放到每张隐藏沙虫图片的随机位置。若至少 70% 的测试图片被判为“非沙虫”，才返回 flag。

因此目标是训练一个跨图片、跨位置和跨旋转仍有效的定向通用对抗补丁，而不是针对单张图片过拟合。

## 解题过程

### 复现服务的随机变换

服务先把每张测试图缩放到 224×224。补丁为 $3\times40\times40$，每次随机旋转并放到合法位置，再通过二值掩码覆盖原图：

```python
def apply_patch(patch, image_size=(3, 224, 224)):
    canvas = np.zeros(image_size)
    rotation = np.random.choice(4)
    patch = np.rot90(patch, rotation, axes=(1, 2)).copy()

    x = np.random.randint(0, image_size[1] - patch.shape[1])
    y = np.random.randint(0, image_size[2] - patch.shape[2])
    canvas[:, x:x + 40, y:y + 40] = patch

    mask = (canvas != 0).astype(np.float32)
    return canvas, mask, x, y
```

训练时必须保留这些随机变换。若始终把补丁贴在固定角落，得到的只是位置相关扰动，几乎无法通过远端随机放置测试。

### 对每张训练图做定向梯度上升

初始化随机补丁，在一批沙虫图片上循环。对当前图片随机贴入补丁后，让合成图可求导，并最大化

$$
\log\sigma(-f(x_{\text{patched}})),
$$

即提高目标类别“非沙虫”的概率：

```python
patched = Variable(patched.data, requires_grad=True)
output = model(patched)
target_score = torch.nn.functional.logsigmoid(-output)
target_score.backward()

patch_grad = patched.grad.clone()
applied_patch = torch.clamp(
    applied_patch + learning_rate * patch_grad,
    min=0,
    max=1,
)
```

只把掩码覆盖区域裁回 40×40，作为下一张图的起点。官方脚本在单张图片上最多更新 100 次，或直到目标类别概率超过 0.9；随后把同一补丁带到下一张训练图。多轮遍历使补丁逐渐吸收不同图片、裁剪、翻转、位置和旋转下都有效的特征，这相当于对变换分布求期望的近似训练。

### 用独立图片选择最佳补丁

每轮结束后分别统计训练集和留出的测试集攻击成功率，仅在测试成功率提高时保存
`patch_best.png`。远端阈值是 70%，本地最好留出明显余量，避免隐藏图片分布和随机旋转造成波动。

服务要求的不是 PNG 文件本身，而是按行排列的 4800 个原始 RGB 字节。提交前应强制转成 RGB、确认尺寸，再 Base64 编码：

```python
import base64
import numpy as np
from PIL import Image

patch = np.asarray(
    Image.open("patch_best.png").convert("RGB"),
    dtype=np.uint8,
)
assert patch.shape == (40, 40, 3)
payload = base64.b64encode(patch.tobytes()).decode()
print(payload)
```

服务按 `(40, 40, 3)` 重塑后再转成 CHW，因此不能直接编码 PNG 压缩文件内容。提交合格补丁后得到：

```text
UMDCTF{sandworms_love_adversarial_patches}
```

## 方法总结

- 核心技巧：在多张图片和随机几何变换上迭代优化同一个定向通用对抗补丁。
- 识别信号：输入不是整张可控图片，而是会被随机旋转、随机放置到隐藏样本上的固定尺寸 patch 时，应使用变换期望训练而非单图梯度攻击。
- 复用要点：训练、验证和服务端的数据布局必须一致；特别要区分 PNG 文件字节与解码后的 RGB 原始字节，并用独立样本验证泛化成功率。
