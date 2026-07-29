# FaILProof

## 题目简述

服务生成一条 SHA-256 哈希链作为“公钥”，把 flag 按 32 字节分块。每个密文分量是 `happiness(pubkey[i] & block)`，同时把生成公钥所需的 16 字节 secret 直接发给客户端。

`happiness` 看似特殊，实际等于整数二进制表示中的 `1` 位数量。于是每个密文分量都是已知公钥位与未知明文位的整数点积，整题可以转成 0/1 线性约束求解。

## 解题过程

函数定义为：

```python
def happiness(x):
    return x - sum(
        x >> i
        for i in range(1, x.bit_length())
    )
```

对任意非负整数 $x$，有恒等式：

$$
x-\sum_{i\ge1}\left\lfloor\frac{x}{2^i}\right\rfloor=\operatorname{popcount}(x).
$$

若把第 $i$ 个 256 位公钥写成比特向量 $a_i$，明文块写成 $x\in\{0,1\}^{256}$，则：

$$
c_i=\operatorname{popcount}(a_i\mathbin{\&}x)=\sum_{j=0}^{255}a_{i,j}x_j.
$$

这里的等式在普通整数上成立，不是模 2 方程；密文值保留了相交比特的总数。

服务公开 `secret.hex()`，所以可以重建 SHA-256 哈希链：

```python
def gen_pubkey(secret):
    state = hashlib.sha256(secret).digest()
    result = []
    for _ in range(32 * 4):
        result.append(int.from_bytes(state, "big"))
        state = hashlib.sha256(state).digest()
    return result
```

把 128 个公钥整数都展开成 256 位，建立布尔变量并交给 CP-SAT：

```python
model = cp_model.CpModel()
x = [model.NewBoolVar(f"x{j}") for j in range(256)]

for row, value in zip(bit_rows, ciphertext_block):
    model.Add(
        sum(coef * bit for coef, bit in zip(row, x))
        == value
    )
```

虽然方程数少于变量数，但每个整数点积可取多个值，信息量远高于一个 GF(2) 方程，随机实例通常足以唯一约束出可行的 256 位块。官方脚本为每个块设置 5 秒上限；若某实例搜索较慢，就重新连接，让服务用新 secret 生成另一组随机方程。

求出比特后按大端序转回 32 字节，依次拼接并去掉 NUL 填充：

```text
SEKAI{w3ll_1_gu355_y0u_c4n_4lw4y5_4sk_f0r_m0r3_3qu4t10n5_wh3n_n0_0n3s_l00k1ng}
```

## 方法总结

位运算外壳不等于非线性密码。先化简 `happiness`，就会发现密文只是已知 0/1 系数与明文比特的整数线性方程。求解时必须保留整数计数，若错误地对 2 取模会丢失大量信息。对于可反复请求的新实例，超时后换一组随机系数也是合理的求解策略。
