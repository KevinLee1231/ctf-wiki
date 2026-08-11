# pressing buttons

## 题目简述

这是一个 iOS UI 游戏，但 flag 不依赖权限、组件或系统沙箱。应用内嵌 74 个五按钮排列；每关必须按 `LEVELS[level][0..4]` 的顺序点击。成功一关后，`ptoc` 把这个长度为 5 的排列转换为一个字符并附加到 flag 文本。因此主要障碍是从 Objective-C 逻辑恢复排列编码，而不是移动平台安全机制，归入 Reverse。

## 解题过程

### 识别排列的 Lehmer 编码

`ptoc` 维护 `unused[5]`，逐位统计当前排列元素左侧仍未使用的较小元素数，并乘以 `tgamma(5-i)`。这里 $\Gamma(5-i)=(4-i)!$，所以：

$$r=\sum_{i=0}^{3} c_i(4-i)!$$

即标准的 Lehmer rank，范围为 $0\ldots119$。若 rank 小于 10，程序加 120，使其对应 ASCII 120 至 129；其余 rank 直接转为 `char`。`doPress` 在每五次正确按键后调用该函数，74 关完成即显示 flag。

### 离线还原

无需真的点击 370 次。官方 solver 将所有 `LEVELS` 按每 5 字节分组，对每个排列计算上述 rank，再对小值作可打印字符补偿，顺序拼接即可。生成脚本也提供反向关系：对每个原始 flag byte 取 `byte % 120`，将 factoradic 数解成相应排列，说明排列表就是 flag 的逐字符编码。

伪代码如下：

```python
for permutation in levels:
    rank = lehmer_rank(permutation)
    chars.append(chr(rank + 120 if rank < 10 else rank))
```

### 验证

题目配置中给出的输出是 `DUCTF{y0u_ar3_g00d_at_pr3ssing_butt0ns_z8y2hzjbx0y7xy19alewp8z9x01pvzq9xy}`。本文没有安装 IPA 或自动点击 UI；控制器源码、生成器和官方 solver 对编码规则一致。

## 方法总结

- 核心技巧：把“按钮顺序”识别为排列，而不是暴力交互；Lehmer/factoradic 编码能把 $n!$ 个排列压缩为整数。
- 识别信号：固定长度排列、`unused` 数组、逐位计数和阶乘/`tgamma` 调用，通常就是排列 rank。
- 复用要点：iOS/Android 载体本身不足以归类 mobile；若只需恢复 app 内的普通算法或状态表，应保留在 Reverse。
