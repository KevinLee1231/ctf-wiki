# SUCTF2026-thief

## 题目简述

题目给出一个迁移学习模型的基础权重 `model_base.pth`，并开放两个接口：`/predict` 接收输入张量并返回完整的 256 维输出，`/flag` 加载选手上传的 `state_dict` 后比较模型参数。网络结构为：

```text
Input(1×32×32)
 -> 左侧、上侧各补 2 个 0
 -> Conv2d(1, 1, 3×3, stride=1)
 -> Conv2d(1, 1, 2×2, stride=2)
 -> Flatten(256)
 -> Linear(256, 256)
```

迁移学习只更新了最后的线性分类层，卷积特征提取器保持不变。对仓库中的基础模型与部署模型逐个比较 PyTorch ZIP storage，可以确认 `conv.weight`、`conv.bias`、`conv1.weight`、`conv1.bias` 完全相同，只有 `linear.weight` 和 `linear.bias` 发生变化。因此可以用本地已知卷积层构造满秩特征输入，再从黑盒输出恢复远程线性层。

仓库中存在三种 `/flag` 校验版本：选手附件用 `elif param.dim() == 1`，会漏掉四维卷积核；归档部署源码改成 `else`，会把卷积核也按 `0.005` 阈值检查；官方 WP 则展示了统一 `0.01` 阈值。由于基础模型和远程模型的卷积参数本来就一致，预期解法不依赖漏检漏洞，能跨这些版本成立。

## 解题过程

### 审计服务与模型

服务使用

```python
torch.load(model_file, weights_only=True, map_location=device)
```

加载上传权重，因此常规 pickle 反序列化 RCE 不是主线。真正有价值的是 `/predict` 返回未经截断的完整输出：

```python
with torch.no_grad():
    outputs = model(torch.tensor(image_data).to(device))
return jsonify({"prediction": outputs.tolist()})
```

设已知卷积部分输出的特征为 $z\in\mathbb{R}^{256}$，远程线性层参数为 $W,b$，则

$$
y=Wz+b.
$$

网络中没有激活函数；只要构造 256 组线性无关的特征差分，就能直接解出 $W$ 和 $b$。

### 官方预期路线：反向构造特征基向量

官方 WP 利用基础模型中卷积核的具体数值，从期望的线性层输入反向构造原始图像。

第二层是 `2×2, stride=2` 卷积。每个输出位置对应互不重叠的 `2×2` 窗口，而该核左上角权重恰为 `0.5`。要让某个输出位置取目标值，只需在对应窗口左上角写入校正后的两倍数值，其他三个位置置 0。

第一层在输入左侧和上侧各补 2 个 0，再使用 `3×3, stride=1` 卷积，输入与输出空间尺寸同为 `32×32`。按从左到右、从上到下的顺序递推时，当前方程中只有一个尚未确定的新输入元素；除以卷积核对应的非零系数，即可逐点反解出原始 `32×32` 输入。

利用该构造器，先令目标特征 $z=0$，查询得到

$$
b'=Wz+b=b.
$$

然后依次令 $z=e_i$，每次查询结果减去基线，直接得到 $W$ 的第 $i$ 列：

$$
W_{:,i}=f(\operatorname{invert\_feature}(e_i))-f(\operatorname{invert\_feature}(0)).
$$

该方法需要 257 次 `/predict` 查询，前提是基础模型中的卷积核允许稳定地执行上述逐层构造。

### 更通用路线：寻找满秩特征矩阵

总 PDF 中的参赛队解法不显式逆卷积，而是利用已知卷积层在本地筛选查询图像。

先对全零图求出本地基线特征 $z_0$ 和远端基线输出 $y_0$。随机生成 256 张 `32×32` 图像，使用基础模型的卷积层计算特征差分并按列组成

$$
Z=[z_1-z_0,\ldots,z_{256}-z_0].
$$

若 `rank(Z) < 256`，就重新生成一组，直到 $Z$ 可逆。对同一组图像查询远端，组成

$$
Y=[y_1-y_0,\ldots,y_{256}-y_0].
$$

由 $Y=WZ$ 可得：

$$
W=YZ^{-1},\qquad b=y_0-Wz_0.
$$

核心代码如下：

```python
def build_query_set(feature, seed=12345):
    rng = np.random.default_rng(seed)
    zero = np.zeros((32, 32), dtype=np.float64)
    z0 = feature(zero)
    while True:
        images = rng.integers(-2, 3, size=(256, 32, 32)).astype(np.float64)
        Z = np.stack([feature(image) - z0 for image in images], axis=1)
        if np.linalg.matrix_rank(Z) == 256:
            return images, Z, z0

images, Z, z0 = build_query_set(local_feature)
y0 = remote_predict(np.zeros((1, 32, 32)))
Y = np.stack([remote_predict(image[None, :, :]) - y0 for image in images], axis=1)
W = np.linalg.solve(Z.T, Y.T).T
b = y0 - W @ z0
```

这种方法同样只需 257 次查询，不要求逐层卷积严格可逆，只要求已知特征提取器的像空间能生成 256 个线性无关方向。查询可以并发，但应限制并发数并保留索引，防止响应乱序造成列错位。

### 组装候选模型

将恢复出的线性层写回基础模型，同时原样保留四组卷积参数：

```python
base = torch.load("model_base.pth", map_location="cpu", weights_only=True)
base["linear.weight"] = torch.tensor(W, dtype=torch.float32)
base["linear.bias"] = torch.tensor(b, dtype=torch.float32)
torch.save(base, "candidate_recovered.pth")
```

没有 PyTorch 时也可以直接处理 `.pth` 的 ZIP 容器：本题 storage `data/0`、`data/1` 分别是线性权重和偏置的小端 `float32` 字节；复制其它条目，只替换这两个 storage 即可。提交前应重新加载候选模型，并用未参与求解的随机图像检查本地输出与远端 `/predict` 的最大误差满足当前服务阈值。

最后把候选权重 Base64 编码提交：

```python
payload = base64.b64encode(Path("candidate_recovered.pth").read_bytes()).decode()
response = requests.post("http://<target>/flag", json={"model": payload})
```

参赛队在比赛服务上恢复成功，得到：

```text
SUCTF{n0t_4ll_h1st0ry_t3lls_th3_truth_6a4e2b8d}
```

## 方法总结

- 核心技巧：利用迁移学习中保持不变的特征提取器，把完整黑盒输出转化为线性方程，恢复最后一层权重和偏置。
- 识别信号：附件提供基础模型，远端只训练分类头，推理接口返回完整向量，且网络最后一层是 `Linear`。
- 复用要点：先验证哪些 storage 真正保持不变，不要把某个校验分支缺陷当成跨版本事实；既可逆向构造特征基向量，也可本地筛选满秩特征矩阵。求解后必须用独立样本检查数值误差和列顺序。
