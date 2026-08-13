# GreyCTF2022 - Data Degeneration

## 题目简述

服务给出 800 个一维样本，它们来自三个方差为 1、均值未知的高斯分布。需要估计三个均值，检查器要求与真值配对后的总误差小于 `0.05`。这是无监督混合模型参数估计问题。

## 解题过程

建立三成分 Gaussian Mixture Model。方差已知，可只更新每个成分的混合权重和均值。E 步计算样本属于每个成分的后验责任度：

$$r_{ik}=\frac{\pi_k\exp(-(x_i-\mu_k)^2/2)}{\sum_j\pi_j\exp(-(x_i-\mu_j)^2/2)}.$$

M 步更新：

$$\mu_k=\frac{\sum_i r_{ik}x_i}{\sum_i r_{ik}},\qquad
\pi_k=\frac{1}{n}\sum_i r_{ik}.$$

```python
means = initialize_from_quantiles(samples, 3)
weights = [1/3] * 3
for _ in range(200):
    resp = expectation(samples, means, weights, sigma=1.0)
    means, weights = maximization(samples, resp)
print(*sorted(means))
```

对初值多运行几次并选择对数似然最高的结果，排序后提交，与服务内部均值完成无序匹配，返回：

```text
grey{3m_iS_bL4cK_mAg1C}
```

## 方法总结

混合模型存在标签交换，提交前应按检查器的匹配方式排序或枚举排列。EM 只保证收敛到局部最优，因此用分位数初始化、多次重启和对数似然比较比单次随机初始化稳健。
