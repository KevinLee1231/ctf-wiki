# GreyCTF2022 - Slow Down

## 题目简述

服务把用户提交的大量整数插入 C++ `std::unordered_map`，并以运行时间是否超过 5 秒作为成功条件。目标环境使用的 libstdc++ 选择素数 42043 作为桶数；构造大量同余键可把哈希表退化为长链。

## 解题过程

对整数键，默认哈希基本就是整数本身，桶索引近似为

$$bucket(k)=k\bmod 42043.$$

因此提交 `0, 42043, 2*42043, ...`，所有键进入同一个桶。插入第 $i$ 个元素时需要在线性链中比较约 $i$ 次，总复杂度从期望 $O(n)$ 退化为 $O(n^2)$。

```python
bucket_count = 42043
payload = [i * bucket_count for i in range(required_count)]
send_numbers(payload)
```

数量要足以超过 5 秒，但仍满足输入个数和整数范围。按服务环境的桶数生成碰撞键后，计时检查通过并返回：

```text
grey{h4sHmaP5_r_tH3_k1nG_of_dAt4_strUcTuRe5}
```

## 方法总结

哈希表平均复杂度依赖输入分布。攻击者可控键且哈希函数无随机化时，集中碰撞会造成算法复杂度拒绝服务。桶数与 rehash 策略属于实现细节，必须在目标 libstdc++ 版本和插入规模上核验，不能跨环境硬编码后直接假定成立。
