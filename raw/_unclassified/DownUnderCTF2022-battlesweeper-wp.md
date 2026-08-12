# DownUnderCTF 2022 battlesweeper Writeup

## 题目简述

服务在 $6\times4$ 棋盘的 24 个位置中随机选 3 个目标。每次必须猜 3 个位置，反馈三个计数：精确命中、与某个目标 Chebyshev 距离为 1、与某个目标距离为 2。需要连续完成 60 局，平均猜测次数低于 4.7，并在 120 秒内结束。

本题的决定性主障碍是纯组合决策策略，无法稳定映射到 14 个安全技术方向，因此暂存 `_unclassified`，而不是机械归入 crypto 或 reverse。

## 解题过程

所有可能目标只有
$\binom{24}{3}=2024$
种，可以始终维护一个与历史反馈一致的候选集合。距离和反馈必须完全复刻服务端的顺序：先移除精确命中的猜测，再统计距离 1，最后统计距离 2，避免一个猜测被重复计数。

```python
from collections import Counter
from itertools import combinations

cells = [(x, y) for y in range(4) for x in range(6)]
all_targets = list(combinations(cells, 3))

def distance(a, b):
    return max(abs(a[0] - b[0]), abs(a[1] - b[1]))

def feedback(target, guess):
    remaining = set(guess)
    exact = {g for g in remaining if g in target}
    remaining -= exact
    near1 = {g for g in remaining
             if any(distance(g, t) == 1 for t in target)}
    remaining -= near1
    near2 = {g for g in remaining
             if any(distance(g, t) == 2 for t in target)}
    return len(exact), len(near1), len(near2)

def prune(candidates, guess, observed):
    return [t for t in candidates if feedback(t, guess) == observed]
```

选择下一次猜测时，枚举候选三元组，并按它可能产生的反馈把当前集合分桶。若桶大小为 $n_1,n_2,\ldots$，反馈后的期望候选数为
$\frac{\sum_i n_i^2}{\sum_i n_i}$；选择该值最小的猜测就是一步信息增益策略：

```python
def score(guess, candidates):
    buckets = Counter(feedback(target, guess) for target in candidates)
    return sum(n * n for n in buckets.values()) / len(candidates)

def best_guess(candidates):
    return min(candidates, key=lambda g: score(g, candidates))
```

完整枚举第一步较耗时，可以离线预计算。官方 solver 得到的固定首猜是：

```text
A0 F0 A1
```

第二步也按首轮剪枝后的候选数预计算成表；从第三步起候选集已足够小，可现场调用 `best_guess`。每次收到 `(3,0,0)` 即进入下一局。该策略偶尔会因随机目标分布使平均值略高，因此官方脚本在未达 4.7 时重新连接再跑一组 60 局。

成功输出：

```text
DUCTF{gg_gu3ss1ng_g0d_8bdaf1232a}
```

## 方法总结

这是有限候选空间上的自适应决策树问题。核心不是猜中概率最高的三个格，而是让所有可能反馈尽量均匀分割候选集；$\sum n_i^2/N$ 正好度量一次猜测后的期望剩余规模。预计算前两步解决时间限制，后续动态求最优则兼顾不同反馈路径。实现时最常见错误是没有按服务端先后顺序排除已计入距离 0/1 的猜测。
