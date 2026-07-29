# Literal Eval

## 题目简述

题目实现了基于 Winternitz One-Time Signature（WOTS）和 Merkle 树的多消息签名服务。每个 Merkle 叶子对应一把一次性签名密钥，服务端试图用 `used` 集合保证叶子不被重复使用。

漏洞来自 Python 容器语义：接口只检查提交对象的长度和迭代出的索引是否唯一，却没有要求它必须是列表。传入包含 255 个键、但所有值均为 0 的字典，可以让校验看到 255 个唯一键，而后续按下标访问时始终取得叶子 0。

## 解题过程

### 1. 构造索引类型混淆

官方载荷使用：

```python
inds = {i: 0 for i in range(255)}
```

对该对象：

```python
len(inds) == 255
len(set(inds)) == 255
```

因为迭代字典得到的是键 $0\ldots254$，唯一性检查通过，`used.update(inds)` 记录的也是这些键。

但签名循环使用整数下标：

```python
leaf_index = inds[i]
```

字典中每个键对应的值都是 0，因此 255 条不同消息全部用 WOTS 叶子 0 签名，并返回相同的 Merkle 认证路径。

### 2. 收集足够多的一次性签名

WOTS 对消息摘要的每个基 $w$ 数位分别给出哈希链上的一个中间值。设第 $j$ 个数位为 $d_j$，签名暴露：

$$
\sigma_j=H^{d_j}(sk_j).
$$

若已经拿到某个数位不大于目标数位的签名值，就可以继续向前哈希：

$$
H^{d_j^\star-d_j}(\sigma_j)
 =H^{d_j^\star}(sk_j).
$$

官方解法收集 255 个随机消息签名。对目标消息 `Give me the flag` 的每个 WOTS 坐标，从样本中挑选数位合适的签名分量，并继续哈希到目标位置。

### 3. 拼接目标签名

伪造流程如下：

```python
for j, target_digit in enumerate(target_digits):
    for sig, digits in samples:
        if digits[j] <= target_digit:
            forged[j] = hash_n(
                sig[j],
                target_digit - digits[j]
            )
            break
```

所有分量都来自同一叶子 0 的私钥，因此可以跨消息拼接；Merkle 根验证只关心最终恢复出的叶子公钥。认证路径也直接复用任一叶子 0 样本。

将伪造的 WOTS 签名、叶子索引 0 和该认证路径提交给目标消息，服务端验证通过并返回 flag。

## 方法总结

一次性签名重复使用会暴露同一组哈希链上的多个位置，收集足够多样本后即可为新摘要逐坐标拼出签名。这里真正触发重复使用的是 Python 的“可迭代对象”和“可下标映射”被混为同一接口。

防护时应先做严格类型与范围规范化，例如：

```python
if not isinstance(inds, list):
    reject()
if any(type(i) is not int for i in inds):
    reject()
```

之后所有唯一性检查、使用记录和签名访问都必须基于同一个规范化列表，不能分别依赖迭代语义与 `obj[i]` 语义。
