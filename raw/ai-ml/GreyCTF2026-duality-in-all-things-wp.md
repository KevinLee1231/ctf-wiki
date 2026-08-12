# Duality in All Things

## 题目简述

附件是一个被裁剪的线性 `sklearn.svm.SVC` 双形式工件，只公开 `support_vectors_`、`dual_coef_`、`intercept_` 和惩罚参数 `C`，省去了通常会直接给出方向的 `coef_` 与原训练行号。flag 没有保存在文本字段中，而是编码在部分 support vector 的 soft-margin slack 值带中；`verify.py` 使用 SHA-256 验证最终字符串。

线性 SVM 的 dual 系数仍足以恢复 primal 分隔超平面。题目把 bitstream 放在 $α_i=C$ 的有界 support vectors 上，并保留了 `support_vectors_` 与 `dual_coef_` 的逐行对齐和顺序。关键能力是利用 SVM 对偶性与 KKT/slack 关系从模型工件恢复隐藏信息，故归入 `ai-ml`。

## 解题过程

### 从对偶参数重建判别函数

对二分类线性 SVM，有

$$
w=\sum_i\alpha_i y_i x_i.
$$

scikit-learn 的二分类 `dual_coef_[0]` 已保存带符号的量 $α_i y_i$，因此不需要原训练标签，也可以直接计算：

```python
signed = artifact.dual_coef_[0]
support_vectors = artifact.support_vectors_
w = signed @ support_vectors
b = artifact.intercept_[0]

labels = np.sign(signed)
margins = labels * (support_vectors @ w + b)
slack = np.maximum(0.0, 1.0 - margins)
```

这里 `labels` 是 support vector 的 $y_i$，而 `abs(signed)` 是 $α_i$。这解释了为何删去 `coef_` 并不构成信息边界：dual 表示本身保留了重构 $w$ 所需的全部相乘项。

### 用有界 support vector 找到比特

soft-margin dual 变量满足 $0\le\alpha_i\le C$。当 $α_i=C$ 时，该点是处于上界的 bounded support vector；其 margin violation 用

$$
\xi_i=\max\left(0,1-y_i(w^Tx_i+b)\right)
$$

表示。筛出这批行后，slack 不再连续散乱，而分成两个明显的一维数值带。无需硬编码生成参数：将 slack 排序，取相邻值最大间隙的中点为阈值，低带映射为 `0`、高带映射为 `1`：

```python
alpha = np.abs(signed)
bounded = np.isclose(alpha, artifact.C, rtol=1e-5, atol=1e-6)
slacks = slack[bounded]                 # 保留原工件行序
ordered = np.sort(slacks)
split = np.argmax(np.diff(ordered))
threshold = (ordered[split] + ordered[split + 1]) / 2
bits = (slacks > threshold).astype(np.uint8)
```

不能按 slack 大小重新排序后再读 bit：分类器对 support-vector 行的共同置换不敏感，但挑战编码要求的是工件原行序。mask 会保持这条公开的行序。

### 解析自校验 bitstream

每八个比特按大端拼为一个字节。得到的数据不是直接盲猜 UTF-8，而有固定的完整性结构：`SVSLACK\x00`、两个字节的大端长度、flag UTF-8 字节和四字节 CRC32。以下解析同时验证两类误差：slack 带切分错误或行序错误都会破坏 magic 或 CRC。

```python
data = bits_to_bytes(bits)
assert data.startswith(b"SVSLACK\x00")
pos = len(b"SVSLACK\x00")
length = int.from_bytes(data[pos:pos + 2], "big")
flag = data[pos + 2:pos + 2 + length]
crc = int.from_bytes(data[pos + 2 + length:pos + 6 + length], "big")
assert zlib.crc32(flag) & 0xffffffff == crc
print(flag.decode())
```

运行官方求解器并把输出送进摘要验证器即可完成闭环：

```bash
python solve/solve.py
python dist/verify.py "$(python solve/solve.py)"
```

恢复结果为 `grey{du4l_0pt1m1z4t10n_l3ft_th3_supp0rt_v3ct0rs_b3h1nd}`，验证器输出 `correct`。

## 方法总结

- 核心技巧：线性 SVM 的 dual 系数 $α_i y_i$ 与 support vector 可完整重建 primal 权重，再用有界 $α_i=C$ 点的 slack 恢复隐写 bitstream。
- 识别信号：模型工件只给 `support_vectors_`、`dual_coef_`、`intercept_`、`C`，且题目暗示 duality、regularization 或 slack 时，应优先检查 KKT 类别和 margin residual，而不是把缺失 `coef_` 当作阻断。
- 复用要点：对顺序编码，筛选 mask 必须保持公开数组原顺序；用 magic、长度和 CRC 等自描述格式验证阈值划分，避免把任意双峰数值误读为 payload。
