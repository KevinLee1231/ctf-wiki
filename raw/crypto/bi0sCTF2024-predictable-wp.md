# bi0sCTF 2024 - Predictable

## 题目简述

服务实现了一个仿 Dual EC DRBG 的椭圆曲线伪随机数生成器。初始化时随机选择曲线点 $P$ 和秘密标量 $d$，令 $Q=dP$；每轮先用 $P$ 更新内部状态，再输出 $Q$ 乘状态所得点横坐标的低 240 位。正常情况下，从截断输出预测下一轮需要知道 $P,Q$ 之间的陷门关系。

漏洞在于服务使用带人工延时的 double-and-add 计算 $Q=dP$，并实时输出进度。每一位的耗时直接泄漏秘密标量 $d$。恢复 $d$ 后，截断输出只隐藏 16 个高位，可以枚举补全点并利用 $e=d^{-1}\bmod n$ 反推内部状态。

## 解题过程

### 从耗时恢复 $d$

标量乘法从最低位开始处理 $d$：

```python
while k > 0:
    if k & 1:
        sleep(0.3 * mul)
        R = Q if R is None else R + Q
    else:
        sleep(0.05 * mul)
    Q = Q + Q
    print(progress, end="\r")
    k >>= 1
```

每收到一次回车符就记录相邻进度输出的时间差。长延时表示当前最低位为 1，短延时表示 0。取两类耗时之间的阈值进行二值化，再反转比特顺序，便得到初始化时使用的标量 $d$：

```python
threshold = sum(delays) / len(delays)
lsb_first = [1 if t > threshold else 0 for t in delays]
d = int("".join(map(str, lsb_first[::-1])), 2)
assert d * P == Q
```

最后的点乘断言很重要。网络抖动可能使靠近阈值的样本被误判；若断言失败，应对耗时聚类或重试，而不是继续使用错误陷门。

### 补全第一份截断输出

记一轮更新后的状态为 $s_1$，完整输出点为

$$
R_1=s_1Q=s_1dP.
$$

服务只返回 $x(R_1)$ 的低 240 位。所选曲线是 192 位或 256 位曲线；实际 solver 对 256 位情形枚举缺失的 16 个高位，对每个候选横坐标

$$
x_i=r_1+i2^{240},\qquad 0\le i<2^{16}
$$

尝试 `lift_x`。能提升到曲线上的横坐标对应两个互为相反数的点，二者都需检查。

令

$$
e=d^{-1}\pmod n.
$$

则对正确的点有

$$
eR_1=e(s_1dP)=s_1P.
$$

其横坐标正是下一轮更新后的状态 $s_2$。用第二份公开输出验证候选：

$$
\operatorname{low}_{240}\left(x(s_2Q)\right)=r_2.
$$

对应实现如下：

```python
for high in range(1 << 16):
    x = (high << 240) | r1
    for candidate in lift_x(x, both_signs=True):
        seed2 = (e * candidate).x
        if ((seed2 * Q).x & ((1 << 240) - 1)) == r2:
            break
```

### 预测下一轮

找到 $s_2$ 后，完全照抄服务端状态转移：

```python
seed3 = (seed2 * P).x
prediction = (seed3 * Q).x & ((1 << 240) - 1)
```

提交 `prediction` 即可通过检查。这里使用第二个输出仅用于筛选补全候选，提交的是第三个输出。

这一思路对应 Dual EC 后门的核心：若攻击者知道 $Q=dP$ 中的 $d$，便可用 $d^{-1}$ 把输出点映回状态更新点；输出截断只会增加一个可枚举的候选空间。相关背景可参阅 [Dual EC 实际可利用性研究](https://eprint.iacr.org/2015/767)，但本题所需的陷门关系、补全范围和验证方法已在上文完整给出。

## 方法总结

本题先通过时间侧信道泄漏 $P,Q$ 间的离散对数关系，再利用该关系攻击截断的 Dual EC 输出。double-and-add 的分支延时泄漏的是从低位到高位的标量位；恢复后必须用 $dP=Q$ 校验。随后枚举仅缺失的 16 个高位、提升曲线点、乘以 $d^{-1}$ 并用第二个输出筛选，即可得到状态并预测下一轮。
