# Rational

## 题目简述

题目用自定义 `Rational` 类型重写了类似 ERC-4626 的金库换算。Setup 往金库存入 $5000$ GREY，玩家可领取 $1000$ GREY；目标是取回总共 $6000$ GREY。

自定义零值编码是 `(0, 0)`。`sub(x, y)` 在 $y=(0,0)$ 时仍套用通分公式，导致任意 $x$ 减零都错误地变成 `(0,0)`，从而可以清空金库的总份额状态。

## 解题过程

`Rational` 把分子放在高 $128$ 位、分母放在低 $128$ 位；零被规范化为整字 `0`，也就是分子和分母都为零。减法实现为：

$$
\frac{a}{b}-\frac{c}{d}=\frac{ad-cb}{bd}
$$

当 $y=0/0$ 时，代码没有直接返回 $x$，而是计算：

$$
\frac{a\cdot0-0\cdot b}{b\cdot0}=\frac00
$$

`redeem(0)` 会从玩家份额和 `totalShares` 中都减去这个特殊零值，因此 `totalShares` 被错误清成零，而金库中的 $5000$ GREY 仍然存在：

```solidity
setup.claim();
setup.vault().redeem(0);
```

此时 `mint(1)` 进入 `convertToAssets` 的零总份额分支，只需支付 $1$ wei GREY 就能获得 $1$ 份额：

```solidity
setup.grey().approve(address(setup.vault()), 1);
setup.vault().mint(1);
```

现在总份额只有 $1$，但金库资产约为 $5000$ GREY。赎回这 $1$ 份额时，换算比例把金库全部余额都归给攻击者：

```solidity
setup.vault().redeem(1);
```

加上领取的 $1000$ GREY，玩家最终达到 $6000$ GREY，取得：

```text
grey{rational_math_fixes_rounding_67c1b078}
```

## 方法总结

自定义数值类型必须为零设计唯一、封闭且与所有运算兼容的表示。这里的 `(0,0)` 既不是合法分数，又被当作普通操作数参与通分，使“减零不变”这一代数恒等式失效，并进一步破坏了金库的会计不变量。审计数学库时，除一般输入外，应逐一测试零、单位元、相等值相减以及分母为零的边界，再检查这些异常值如何传导到资产与份额状态。
