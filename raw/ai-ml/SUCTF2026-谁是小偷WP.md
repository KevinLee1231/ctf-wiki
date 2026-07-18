# SUCTF2026-谁是小偷

## 题目简述

题目要求从黑盒推理接口恢复一个 PyTorch 模型，并把伪造的 `state_dict` 以 Base64 提交给 `/flag`。线上主路径实际为：

```text
Input(1×1×19×19) -> Conv2d(1, 1, 4×4) -> Flatten(256) -> Linear(256, 256)
```

模型没有激活函数，因而整体是仿射映射。`/predict` 返回完整的 256 维输出；`/flag` 用 `torch.load(..., weights_only=True)` 加载选手模型，并要求各参数与真实参数的逐元素误差不超过 `0.01`。

附件刻意制造了冲突：`attachment/app.py` 把卷积核写成 `100×100`，而一页 briefing 明确提示“附件是热身材料，不是线上完整快照”“先看行为，再看命名”。归档中的部署源码则能确认真实卷积核是 `4×4`。因此解题时应以可重复验证的接口行为、矩阵形状和参数关系为准。

官方 WP 给出的预期捷径还假设可取得与远程模型共享线性层的 `model_base.pth`。需要注意，当前仓库的 `SU_谁是小偷` 目录及其 `attachment.zip` 并未包含该文件，仓库中的官方 `solve_remote.py` 却会直接读取它；所以该捷径只在拿到原始基础模型（或确认可复用前置题模型）时成立。下面同时记录官方捷径和不依赖该文件的完整黑盒恢复法。

## 解题过程

### 确认真实结构

向 `/predict` 发送不同尺寸的输入并观察线性层报错：

- `1×1×115×115` 会在展平后得到 12544 个元素，无法与 `256×256` 线性层相乘；
- `1×1×28×28` 会得到 625 个元素，同样不匹配；
- `1×1×19×19` 可以正常推理。

合法输入经过卷积后必须为 `16×16=256`，步长为 1、无 padding 时满足

$$
19-K+1=16,
$$

因此真实卷积核为 `4×4`。这与部署源码一致，而与附件中的烟雾弹不一致。

### 官方预期路线：利用已知线性层

若已取得基础模型中的 `linear.weight=W` 和 `linear.bias=b`，前向过程为

$$
y=Wz+b,
$$

其中 $z$ 是卷积输出展平后的 256 维向量。官方题解验证了 $W$ 可逆，于是一次推理即可反推出中间层：

$$
z=W^{-1}(y-b).
$$

先提交全零输入，得到 $y_0$。此时卷积输出的 256 个位置都等于同一个偏置：

```python
z0 = np.linalg.inv(W) @ (y0 - b)
conv_bias = z0.mean()
```

再只令输入 `(3, 3)` 为 1，得到 $y_1$，将反推出的 `z1` reshape 为 `16×16` 并减去偏置。PyTorch 的 `Conv2d` 实际执行互相关，因此左上角 `4×4` 响应是卷积核的双轴翻转；再翻转一次即可恢复原核：

```python
z1 = (np.linalg.inv(W) @ (y1 - b)).reshape(16, 16)
delta = z1 - conv_bias
kernel = np.flip(delta[:4, :4], axis=(0, 1))
```

官方复现得到：

```text
conv.weight =
[[-6, -10,  1, -4],
 [ 6,  -1,  8,  8],
 [ 9,  -7,  6, -4],
 [-5,  6,  8, -6]]

conv.bias = 4
```

用另一组非稀疏输入比较本地前向结果和 `/predict` 返回值，可以在提交前验证恢复是否正确，而不应只依赖最终 `/flag` 的成功与否。

### 无基础模型时：恢复整体仿射矩阵

总 PDF 给出的参赛队解法不依赖 `model_base.pth`。令展平输入为 $x\in\mathbb{R}^{361}$，整个网络可写成

$$
y=Tx+B,
$$

其中 $T\in\mathbb{R}^{256\times361}$。一次全零查询得到 $B$，再对 361 个输入位置逐一发送基向量 $e_i$：

```python
B = predict(np.zeros((1, 1, 19, 19), dtype=np.float32))
for i in range(361):
    image = np.zeros((1, 1, 19, 19), dtype=np.float32)
    image.reshape(-1)[i] = 1
    T[:, i] = predict(image) - B
```

`T` 的右零空间维数为 `361-256=105`。在线性层满秩的条件下，$Tx=0$ 等价于该输入经过卷积后的 256 个窗口内积全部为 0。将 105 个零空间向量 reshape 为 `19×19`，枚举其中所有 `4×4` 窗口，把它们堆成关于 16 个卷积核元素的齐次方程组；该方程组最小奇异值对应的右奇异向量就是卷积核，差一个整体比例因子。

本题卷积核是小整数。将奇异向量按元素比例归一并取整，可以得到上面的真实核。随后构造卷积展开矩阵 $P_c\in\mathbb{R}^{256\times361}$，满足

$$
T=W_LP_c,
$$

所以可用伪逆恢复线性层：

$$
W_L=TP_c^+.
$$

恢复出的 `linear.weight` 接近 `[-10, 9]` 内的整数，取整后可作为卷积核比例正确的交叉验证。整体偏置满足

$$
B_m=b_c\sum_k W_L[m,k]+b_{L,m}.
$$

令 $S_m=\sum_kW_L[m,k]$，对 $(S_m,B_m)$ 做线性拟合，斜率约为 4，即 `conv.bias=4`；再由 $b_L=B-b_cS$ 得到 `linear.bias`。

### 构造并提交模型

将四组参数按服务端模型的键名保存：

```python
state_dict = {
    "linear.weight": torch.tensor(W_L, dtype=torch.float32),
    "linear.bias": torch.tensor(b_L, dtype=torch.float32),
    "conv.weight": torch.tensor(kernel, dtype=torch.float32).reshape(1, 1, 4, 4),
    "conv.bias": torch.tensor([conv_bias], dtype=torch.float32),
}

buffer = io.BytesIO()
torch.save(state_dict, buffer)
payload = base64.b64encode(buffer.getvalue()).decode()
requests.post("http://<target>/flag", json={"model": payload})
```

若环境中没有 PyTorch，也可以复用基础模型 ZIP 归档中的线性层 storage，并按 PyTorch ZIP 序列化格式补入卷积核和偏置；官方脚本演示了这种最小归档构造方式。参赛队黑盒路线最终通过了 `0.01` 误差校验，取得 flag：

```text
SUCTF{ch3ck_th3_st4t3_n0t_th3_l0g_5d1f9a6c}
```

## 方法总结

- 核心技巧：利用纯线性网络的可逆中间层泄露，或用 basis query、零空间 SVD 和伪逆完成黑盒模型窃取。
- 识别信号：`Conv2d -> Flatten -> Linear` 中没有激活函数，推理接口返回完整向量，且提交接口逐参数比较模型权重。
- 复用要点：附件、命名与线上行为冲突时先核对形状；有已知可逆线性层时只需极少查询，无该层时再恢复整体仿射矩阵。任何依赖文件都应先确认实际归档中存在，不能把官方脚本的隐含前提当成事实。
