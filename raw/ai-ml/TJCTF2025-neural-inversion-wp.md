# neural-inversion

## 题目简述

`model.npz` 保存了一个两层全连接 sigmoid 网络的参数 `W1, b1, W2, b2` 和输出向量 `y`。flag 的每个 ASCII 码先除以 127 形成输入向量 $x$；两个权重矩阵均为满秩方阵，所以这不是训练或搜索问题，而是对已知网络逐层做代数逆运算。

## 解题过程

生成器的前向过程为

$$
h=\sigma(W_1x+b_1),\qquad
y=\sigma(W_2h+b_2),
$$

其中 $\sigma(z)=1/(1+e^{-z})$。sigmoid 在 $(0,1)$ 上的逆函数是 logit：

$$
\operatorname{logit}(p)=\ln\frac{p}{1-p}.
$$

因此先对 $y$ 取 logit，并解线性方程 $W_2h=\operatorname{logit}(y)-b_2$；再对恢复出的 $h$ 取 logit，解 $W_1x=\operatorname{logit}(h)-b_1$。最后把 $x$ 乘以 127、四舍五入为整数 ASCII。与显式计算矩阵逆相比，`numpy.linalg.solve` 数值上更稳健。

```python
import numpy as np

def logit(p):
    return np.log(p / (1.0 - p))

model = np.load("model.npz")
W1, b1 = model["W1"], model["b1"]
W2, b2 = model["W2"], model["b2"]
y = model["y"]

h = np.linalg.solve(W2, logit(y) - b2)
x = np.linalg.solve(W1, logit(h) - b1)

codes = np.rint(x * 127).astype(int)
assert np.all((0 <= codes) & (codes < 128))
print("".join(chr(c) for c in codes))
```

在发布的模型上运行得到：

```text
tjctf{m0d3l_1nv3rs10n_f9a93qw}
```

## 方法总结

- 核心技巧：从输出端开始，交替应用激活函数的逆和线性方程求解，逐层恢复网络输入。
- 识别信号：输入维度与输出维度相同、权重是满秩方阵、激活函数单调可逆，并且模型同时泄露目标输出。
- 复用要点：实际模型若有 ReLU、量化、降维、噪声或非满秩权重，精确逆解通常不存在；此时才需要优化、约束求解或统计恢复。
