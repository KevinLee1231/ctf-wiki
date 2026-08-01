# Bank Vault

## 题目简述

程序内置一个 84 位 `vector<bool> mark` 和 84 个 `uint32_t keys`。循环从 `mark2` 尾部依次取位，用当前输入字符和 `keys[i]` 的中间字节比较，将比较结果加入 `result`；只有当前标记位为真时才推进输入下标。最后反转 `result` 并要求其等于原始 `mark`。

混淆来自两个方向相反的布尔序列以及条件式输入推进，核心校验本身仍是可逆的逐字节相等判断。

## 解题过程

第 $i$ 轮读取的期望布尔值就是 `mark` 的倒序位：

$$
curr=mark[83-i].
$$

因为 `result` 最后还会反转，所以要通过总校验，原始比较结果也必须等于该 `curr`。当 `curr` 为真时，当前 flag 字符必须等于：

$$
(keys[i]\gg16)\mathbin{\&}0xff.
$$

此时程序才把输入下标加一。因此按相同顺序提取这些中间字节即可：

```python
out = []
for i, key in enumerate(keys):
    curr = mark[-1 - i]
    if curr:
        out.append((key >> 16) & 0xff)

flag = bytes(out)
print(flag)
```

`curr` 为假的轮次要求比较结果为假，生成器中的随机 key 已保证正确 flag 不会意外相等。提取出的字符串为：

```text
byuctf{3v3n_v3ct0rs_4pp34r_1n_m3m0ry}
```

将它交给原程序会输出 `Access granted!`。

## 方法总结

- 核心技巧：把反转后的向量等式换算回每轮所需布尔值，只在推进输入下标的真分支提取关键字节。
- 识别信号：校验器同时维护标记位、结果数组和条件索引时，应先写出索引映射，避免被 `reverse` 与 `pop_back` 干扰。
- 复用要点：不要把每个 key 都当成 flag 字符；先确定哪些比较必须为真，再提取对应字段并回放原校验。
