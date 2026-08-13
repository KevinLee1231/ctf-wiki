# GreyCTF2022 - Root

## 题目简述

服务允许查询一条以秘密 `root` 为根的多项式。将查询点取为 $x=0$ 时，返回的常数项包含 `root` 与一个随机素数因子的乘积；由于 `root` 有明确的小位数上界，分解该常数即可直接筛出答案。

## 解题过程

从生成逻辑可知多项式含因子 $(x-root)$，其余因子常数项的乘积记为 $P$。于是

$$f(0)=-root\cdot P.$$

向服务提交 `0`，取得整数 $v=f(0)$。对 $|v|$ 分解后，枚举不超过题目给定 $2^{70}$ 上界的素因子或因子组合，并用服务的校验接口确认：

```sage
value = abs(f_at_zero)
for prime, exponent in factor(value):
    if prime < 2^70:
        candidates.append(prime)
```

实际数据中目标根可从小因子中唯一识别。提交后服务返回：

```text
grey{The_Answer_To_The_Riddle_Is_"Road"!_ZvBtTpyA4GXuuwjB}
```

## 方法总结

多项式 oracle 不一定需要插值。先尝试 $0$、$1$、$-1$ 等能让表达式退化的特殊点，往往能把隐藏根变成乘法因子。候选筛选应结合位数上界，并通过原多项式或服务检查，而不能仅凭“因子看起来像文本”下结论。
