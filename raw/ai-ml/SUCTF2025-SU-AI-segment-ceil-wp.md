# SU_AI_segment_ceil

## 题目简述

题目要求对细胞显微图像做二值语义分割。服务端随机抽取十张图，加入均匀噪声后以 Base64 PNG 发送；客户端必须在每张图发出后的两秒内返回同尺寸分割掩码。

服务端只把纯黑像素视为前景，并用

$$
\operatorname{score}
=\frac{2\lvert P\cap G\rvert}{\lvert P\rvert+\lvert G\rvert}\times100
$$

评分，其中 $P$ 是预测黑色像素集合，$G$ 是标签黑色像素集合。每张图都必须达到 $78$ 分。这实际上就是以前景集合计算的 Dice 系数。

当前总仓库只留下不足 700 字的官方概述，没有训练集、服务端和官方 `attack` 文件。服务端评分代码、训练日志、完整训练 notebook 与最终 flag 由[公开赛后题解](https://outw.rest/posts/suctf2025-ai-writeups)及其[训练 notebook](https://github.com/OutWrest/blog-handouts/blob/main/suctf2025-ai-writeups/SU_AI_segment_ceil/train.ipynb)补足，正文已归纳解题所需的关键机制。

## 解题过程

### 1. 按服务端规则准备标签

读取训练图和对应 PNG 标签，把标签中 RGB 都为零的像素映射为前景 `1`，其余像素映射为背景 `0`。不要把灰度小于某个阈值直接等同于服务端真值；训练阶段可以做阈值清洗，但最终发回的 PNG 必须显式写成纯黑或纯白：

```python
mask = np.all(label_rgb == 0, axis=2).astype(np.float32)
```

训练/验证划分必须按原始图像划分，然后再做增强，避免同一张图的不同增强版本同时出现在两侧。

### 2. 训练抗噪 U-Net

数据量较少，普通训练容易记住样本而无法适应服务端随机噪声。可采用轻量 U-Net，并在训练时同步增强图像和标签：

- 水平/垂直翻转与 $90^\circ$ 旋转；
- 随机裁剪、缩放后恢复原尺寸；
- 对图像加入与服务端接近的均匀噪声；
- 轻微亮度、对比度变化；
- 标签只使用最近邻插值，避免产生非二值边界。

损失函数使用二元交叉熵与 Dice loss 的组合：

$$
\mathcal L
=\operatorname{BCEWithLogits}(z,G)
+\left(
1-\frac{2\sum \sigma(z)G+\varepsilon}
{\sum \sigma(z)+\sum G+\varepsilon}
\right).
$$

公开复现训练 100 个 epoch，在独立验证集上达到约 $79.76\%$ 的像素准确率，并通过多次在线测试越过 $78$ Dice 门槛。官方概述还指出，推理前使用简单的中值或低通滤波可以削弱均匀噪声；更稳妥的做法是训练时就模拟同分布噪声，滤波只作为轻量预处理。

### 3. 在两秒限制内完成交互

模型应在连接服务前加载到内存并执行一次预热，收到图像后只做解码、一次前向推理和 PNG 编码：

```python
raw = base64.b64decode(received_image)
image = Image.open(io.BytesIO(raw)).convert("RGB")

tensor = preprocess(image).unsqueeze(0).to(device)
with torch.inference_mode():
    probability = torch.sigmoid(model(tensor))[0, 0].cpu().numpy()

foreground = probability >= threshold
result = np.where(foreground[..., None], 0, 255).astype(np.uint8)
result = np.repeat(result, 3, axis=2)

buffer = io.BytesIO()
Image.fromarray(result, "RGB").save(buffer, format="PNG")
sendline(base64.b64encode(buffer.getvalue()))
```

阈值不要固定凭感觉选择，应在验证集上直接以服务端同款 Dice 指标搜索。公开复现通过十张随机图后得到：

```text
SUCTF{Any_help_is_better_than_no_help}
```

## 方法总结

本题的决定性障碍是小样本、加噪条件下的图像分割，而不是传输层或 Base64。先准确复刻服务端“纯黑前景 + Dice 大于 78”的判定，再围绕该指标做数据增强、抗噪训练和阈值选择，远比追求通用像素准确率可靠。两秒限制要求把训练、模型加载和预热全部移到连接之前，在线阶段只保留一次推理流水线。
