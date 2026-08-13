# GreyCTF2022 - Pursuit

## 题目简述

服务在有向加权图上隐藏一个会移动的目标。每轮选手猜节点；失败只说明目标不在该节点，随后目标按转移概率移动。需要在 100 轮内连续命中 5 次，核心是维护位置的后验分布。

## 解题过程

用概率向量 $p_t(v)$ 表示猜测前目标位于节点 $v$ 的概率。选择概率最大的节点 $g$ 作为猜测；若失败，将该节点概率置零并重新归一化：

$$p'_t(v)=\frac{p_t(v)\mathbf 1[v\ne g]}{1-p_t(g)}.$$

随后用图的转移矩阵 $T$ 推进一轮：

$$p_{t+1}(u)=\sum_v p'_t(v)T_{v,u}.$$

```python
guess = max(nodes, key=lambda v: posterior[v])
send_guess(guess)
if miss:
    posterior[guess] = 0.0
    normalize(posterior)
posterior = transition(posterior, graph)
```

若服务在命中后仍移动目标，则沿相同转移更新；若命中重置状态，则按协议重置先验。按仓库服务逻辑维护后验，可在预算内累计连续命中并取得：

```text
grey{wHY_4Re_u_RunN1n9}
```

## 方法总结

这类追踪题不能只按转移图做静态最短路；每次失败也是观测，必须先做贝叶斯条件化再做状态转移。更新顺序、命中后的状态规则和边权归一化都应从服务源码逐项核对。
