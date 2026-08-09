# MaaS

## 题目简述

服务随机生成 16 个大写字母。对每个位置，玩家可查询三次 $(x\ll16)\bmod c$，其中 $c$ 是该位置字符的 ASCII 码；需要利用模运算结果唯一确定字符，再提交完整字符串。

## 解题过程

若查询值为 $x$，响应为 $r$，则：

$$x\cdot2^{16}-r\equiv0\pmod c.$$

因此字符码 $c$ 是 $x\cdot2^{16}-r$ 的因子。对同一位置分别提交 `1`、`3`、`7`，再从大写字母的 ASCII 范围中筛选同时满足三组余数的字符即可。直接建立响应三元组到字符的映射更简洁：

```python
from string import ascii_uppercase

queries = (1, 3, 7)
lookup = {
    tuple((x << 16) % ord(ch) for x in queries): ch
    for ch in ascii_uppercase
}

# 每个位置读取三次响应 r0、r1、r2：
# answer += lookup[(r0, r1, r2)]
```

恢复 16 个随机字母并提交后，服务返回：

```text
n00bz{M0dul0_f7w_1a4d3f5c!}
```

## 方法总结

模运算 oracle 泄露了模数的整除关系。由于候选模数只可能是 26 个大写字母的 ASCII 码，少量不同查询即可消除碰撞，不必对大整数做无界因数分解。
