# Sigma

## 题目简述

附件先把 flag 按每个字符 8 位大端形式转换成比特串，再生成权重：

$$
M_0=2,\qquad M_{i+1}=2M_i+1.
$$

最终只公开子集和

$$
S=\sum_i b_iM_i,\qquad b_i\in\{0,1\}.
$$

这看起来像背包密码，但权重序列没有经过置换或模乘隐藏，原始的超递增结构被直接公开。

## 解题过程

序列前几项为 $2,5,11,23,\ldots$，且 $M_i=3\cdot2^i-1$。任意一项都严格大于此前所有项之和，因此从最大权重向下贪心不会产生歧义：

- 若当前余数不小于 $M_i$，则 $b_i=1$ 并减去 $M_i$。
- 否则 $b_i=0$。
- 得到的是从高权重到低权重的比特，需要反转回原顺序。

因为 flag 以字符“}”结束，其最后一个二进制位是 1，可以从 $S$ 向上生成权重直到下一项超过 $S$，无需预先知道长度：

~~~python
S = 16959882212901931085914486209869339126650130022168939505908957058292634023420380120

weights = []
value = 2
while value <= S:
    weights.append(value)
    value = 2 * value + 1

remaining = S
bits_desc = []
for weight in reversed(weights):
    if remaining >= weight:
        bits_desc.append("1")
        remaining -= weight
    else:
        bits_desc.append("0")

assert remaining == 0
bits = "".join(reversed(bits_desc))
plain = bytes(
    int(bits[i:i + 8], 2)
    for i in range(0, len(bits), 8)
)
print(plain.decode())
~~~

输出为：

~~~text
shellmates{Kn4p54ck_Cryp70_Sys73m}
~~~

## 方法总结

Merkle–Hellman 一类背包方案的安全性依赖隐藏超递增私钥；若直接用超递增序列做公开权重，子集和可以贪心唯一分解。遇到背包密文时，应先检查每个权重是否大于之前权重之和，再决定是否需要格攻击。还要明确“比特顺序”和“权重顺序”，避免恢复出整体反转的字节流。
