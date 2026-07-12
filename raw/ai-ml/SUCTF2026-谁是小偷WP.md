# SUCTF2026-谁是小偷

## 题目简述
题目是一个黑盒神经网络参数恢复题。附件 `app.py` 给出 Flask 服务接口、模型上传/预测逻辑和一个误导性的 `Conv2d(1, 1, (100, 100))`；线上真实行为必须以 `/predict`、`/flag` 的报错和返回为准。真正有价值的信号是 `/predict` 的矩阵维度报错：合法输入为 `19x19`，网络等价于一层 `4x4` 卷积后接 `256 -> 256` 线性层。解法核心不是相信附件静态形状，而是把没有激活函数的模型视为仿射映射来恢复参数。

## 解题过程
这道题考察了对线性神经网络结构的黑盒参数还原。服务端提供了一个简单的两层模型：一层卷积层

和一层全连接层。

### 1.1 确定真实的输入及网络形状

首先下载 app.py 及相关 pdf 的提示信息指出：“如果多条线索互相矛盾，请先核对形状”。在
app.py 中的提示代码给的是 Conv2d(1, 1, (100, 100)) ，这显然是个烟雾弹。

我们可以通过对 /predict 接口发送不同大小的 tensor 并观察报错信息，来推断真实的网络结
构：

- 发送 1x1x115x115 ，报错 1x12544 and 256x256 cannot be multiplied 。

- 发送 1x1x28x28 ，报错 1x625 and 256x256 cannot be multiplied 。

- 只有发送 1x1x19x19 时能够成功进行推理，证明真实的输入形状是 19x19。
在这个尺寸下，全连接层需要输入为 16 × 16 = 256 个特征，因此卷积之后输出的空间尺寸是
16 × 16 。
根据卷积输出计算公式 `19 - K + 1 = 16`，推导出真实卷积核 $W_c$ 的大小为 `4x4`。

### 1.2 网络的纯线性特性与等效转换

题目中所给出的网络不包含任何激活函数（如 ReLU），整个模型只有Conv2d 、Flatten 、
Linear ，是一个纯线性变换可以表示为如下形式：

$$
y = T \cdot x + B
$$

其中 $x$ 是展平后的 `1 x 361` 输入图像，$T$ 是结合卷积和全连接逻辑的 `256 x 361` 等效矩阵，$B$ 是整体偏置，$y$ 是 `1 x 256` 输出。

这就意味着，只要我们输入大量使用 One-Hot 编码（例如只在某一像素位置设为 1，其他均为 0）的
图片数组，就能完美地“解剖”出这个等效矩阵 T ：
1. 请求全 0 图像得偏置：B = predict(zeros) 。
2. 逐像素将 `img[i][j]` 设为 1：`T[:, idx] = predict(e_idx) - B`，即可获得对应输入像素对输出的影响。

### 1.3 SVD 零空间求解卷积层权重 $W_c$

有了 $T$，需要将其拆解成 Conv2d 的参数 $W_c$（`4x4`，16 个参数）和 Linear 的参数 $W_L$（`256x256`）。

从原结构看，对于任意处于零空间的输入 $x_{null} \in N(T)$，都会使得模型的无偏置输出为 0。因为 $W_L$ 通常是满秩的，所以 $T x_{null}=0$ 等价于卷积层在输入 $x_{null}$ 下的输出全为 0。

`256 x 361` 的 $T$ 右零空间可以通过 SVD（`np.linalg.svd`）计算得到，维度为 `361 - 256 = 105`。
我们拿出这105个非平凡全0特征图，它们通过这唯一的一个$4\times4$滤波器滑动时，每一个滑动
窗口都会在滤波器内积下为0：

$$
\sum_{i,j} W_c[i,j] \cdot X_{\text{window}}[i,j] = 0
$$

通过把这些滑动生成的 `4 x 4` 窗口重排成关于 16 个卷积核参数的方程组，再做 SVD 分析，最小奇异值对应的右奇异向量就是真实卷积核 $W_c$ 的参数。此时只能确定比例，即求出的是带未知比例因子 $k$ 的版本。

好在一旦输出后检查它的元素比例，我们会惊喜地发现各个权重的数值比例惊人的齐整且都是小整数
配比。通过除以一个恰当的值后四舍五入，完美揭示出真实的整型 $W_c$ 矩阵。

### 1.4 伪逆计算并还原 $W_L$ 和所有 Bias

在求出准确的 $W_c$ 参数后，就能构造卷积层展开后的线性算子 $P_c \in R^{256 \times 361}$。此时：

$$
T = W_L \cdot P_c
$$

由于 $P_c$ 不一定满秩，直接求它的伪逆（`np.linalg.pinv`）：

$$
W_L = T \cdot P_c^+
$$

计算出的矩阵 $W_L$ 取整后仍然是严格介于 `[-10, 9]` 之间的小整数，这表明 $W_c$ 的比例因子猜对了。

最后来分离偏置（Bias）：

网络总偏置满足：

$$
B_m = b_c \sum_k W_L[m,k] + b_{L,m}
$$

其中 $b_c$ 是单个 `conv.bias`，$b_L$ 是长度 256 的 `linear.bias`。把 $S_m=\sum_k W_L[m,k]$ 视为自变量、整体偏置 $B_m$ 视为因变量，就得到一条线性回归关系。

通过对数据做一次多项式拟合或直接除法分析：

```python
S = WL.sum(axis=1)
slope, intercept = np.polyfit(S, B, 1)
```

发现斜率为接近完美的 4.0 。因此：

- 真实的 `conv.bias`（$b_c$）为 `4.0`。

- 根据 $b_{L,m}=B_m-b_c S_m$，取四舍五入即可提取真实的 `linear.bias`。所有参数和精度提取由此闭环。

### 1.5 构造 Payload 拿 Flag

最后通过 PyTorch 序列化恢复的模型，Base64 编码后 POST 给 /flag 接口：

```python
import requests, torch, base64, io, numpy as np
# 将上面反解出的四大参数装入 state_dict
b = io.BytesIO()
torch.save({
'linear.weight': torch.tensor(W_L_int, dtype=torch.float32),
'linear.bias': torch.tensor(b_L, dtype=torch.float32),
'conv.weight': torch.tensor(W_c_int, dtype=torch.float32).view(1, 1, 4, 4),
'conv.bias': torch.tensor([4.0], dtype=torch.float32)
}, b)

r = requests.post('http://<target>:10002/flag', json={'model':
base64.b64encode(b.getvalue()).decode()})
print(r.json())
```

服务器校验要求 `abs(param - user_param) <= 0.01`，恢复出的模型可以通过判定：
SUCTF{ch3ck_th3_st4t3_n0t_th3_l0g_5d1f9a6c}

## 方法总结
- 核心技巧：线性神经网络黑盒参数恢复
- 识别信号：看到 `Conv2d + Flatten + Linear` 且无激活函数、`/predict` 返回完整向量时，应想到 basis query 恢复整体仿射矩阵。
- 复用要点：附件形状与线上行为冲突时，以报错维度和可推理输入尺寸为准；再用零空间/SVD、伪逆分解出卷积层和线性层。
