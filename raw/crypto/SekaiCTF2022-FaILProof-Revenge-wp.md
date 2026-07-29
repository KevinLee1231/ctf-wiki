# FaILProof Revenge

## 题目简述

Revenge 版仍使用：

```python
happiness(pubkey[i] & block)
```

作为密文，但改用 SHA-512，并在每个 8 字节 flag 分块前后分别加入 24 字节和 32 字节随机数据。每个实际未知块因此长 64 字节，共 512 位；服务提供 256 条整数点积方程。

随机填充阻止直接依赖 flag 格式缩小变量，却没有改变方程本身可由 0/1 约束求解的事实。

## 解题过程

同基础题一样：

$$
\operatorname{happiness}(z)=\operatorname{popcount}(z).
$$

对 512 位未知向量 $x$ 和已知公钥比特行 $a_i$，每个密文值满足：

$$
c_i=\sum_{j=0}^{511}a_{i,j}x_j,
\qquad x_j\in\{0,1\}.
$$

服务把 secret 发给客户端，因此 SHA-512 哈希链可完整重建：

```python
state = hashlib.sha512(secret).digest()
pubkey = []

for _ in range(64 * 4):
    pubkey.append(int.from_bytes(state, "big"))
    state = hashlib.sha512(state).digest()
```

明文打包顺序非常重要：

```python
padded_block = (
    urandom(24)
    + flag_block
    + urandom(32)
)
```

所以求解 512 个布尔变量后，真正需要的是恢复块的字节区间 `[24:32]`。

CP-SAT 模型为：

```python
model = cp_model.CpModel()
x = [model.NewBoolVar(f"x{j}") for j in range(512)]

for public_value, encrypted_value in zip(
    pubkey, encrypted_block
):
    row = [
        int(bit)
        for bit in f"{public_value:0512b}"
    ]
    model.Add(
        sum(a * b for a, b in zip(row, x))
        == encrypted_value
    )
```

设置单实例求解时间上限。如果某个分块在当前随机矩阵上搜索过慢，就重新连接获取另一组 secret 和方程；flag 分块不变，可以分别从不同实例恢复。

将模型解按大端序组成 64 字节，截取中间 8 字节并拼接，最后去掉 NUL 填充：

```python
recovered = bits_to_bytes(solution).rjust(64, b"\0")
flag_part = recovered[24:32]
```

得到：

```text
SEKAI{b00l3n5_4r3_my5t3r10u5_1n_th31r_0wn_ch33ky_w4y5}
```

## 方法总结

随机前后缀增加了未知变量，但没有引入密码学非线性；每条 popcount 仍是普通整数上的线性等式。只要方程携带的总信息足够，0/1 约束求解器可以连同随机填充一起恢复，再按明确布局切出中间 flag。此类题的关键是从源码精确确认比特宽度、字节序和分块偏移。
