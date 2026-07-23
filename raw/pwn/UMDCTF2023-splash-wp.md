# Splash

## 题目简述

题目模拟宝可梦对战：己方 Arceus 只会没有伤害的 Splash，敌方 Magikarp 只有 $1$ 点生命。正常消耗 PP 几乎不可能，因为 Splash 的初始 PP 为 `INT_MAX - 1`，而敌方每回合会造成 $128$ 点伤害。突破点是给 PP 做加法时没有检查有符号整数溢出。

## 解题过程

初始状态为：

```c
int magikarp_health = 1;
int arceus_health = 514;
int splash_pp = INT_MAX - 1;
```

选择背包中的 PP UP 会执行：

```c
splash_pp += 1;
arceus_health -= 128;
```

第一次选择 `BAG` 后，PP 从 `INT_MAX - 1` 变为 `INT_MAX`，生命从
$514$ 变为 $386$。第二次再选择 `BAG`，在题目构建环境的二补码行为下，
PP 绕回到负数，生命变为 $258$。

此时选择 `FIGHT`。判断 `splash_pp > 0` 为假，程序进入没有 PP 的分支，让
Arceus 使用 Struggle。$258 > 514/4$，所以反伤检查不会令己方倒下，而 Magikarp 被击败并调用 `printflag()`。

完整输入序列为：

```text
2
2
1
```

最终得到：

```text
UMDCTF{7H3_M0UN741N_I_SPL4SH_0V3R}
```

## 方法总结

- 核心漏洞是对接近 `INT_MAX` 的有符号计数器连续加一，导致逻辑条件从“极大正数”翻转为负数。
- 利用前要同时计算生命值：溢出需要两回合，之后还必须满足 Struggle 的反伤检查。
- 严格的 C 语言标准把有符号溢出视为未定义行为；这里的结论依赖题目给定编译产物表现，不能机械推广到任意优化配置。
