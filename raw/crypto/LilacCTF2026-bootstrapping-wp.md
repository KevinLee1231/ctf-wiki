# BootsTrapping

## 题目简述

题目使用 Microsoft SEAL/CKKS 生成两个密文文件：`ciphertext.dat` 保存带 flag 的密文，`ciphertext_mask.dat` 保存 mask 密文。源码模拟了 CKKS bootstrapping 中的“升模、同态解密、模约减”过程，但在模约减阶段引入了 32-bit `int` 截断陷阱。

关键参数为：

```cpp
size_t n = 4096;
parms.set_coeff_modulus(CoeffModulus::Create(n, {30, 30, 30}));
```

## 解题过程

### 关键观察

题目模拟的 bootstrapping 过程可以分成三步：

1. 将低模数密文提升到高模数多项式环。
2. 用私钥进行“同态解密”形式的计算，得到中间值：

$$
I(x) = m + e + k \cdot q_{low}
$$

3. 对中间值模 `q_low` 约减，去掉溢出项 `k * q_low`。

漏洞在第三步。源码把 CRT 还原后的大整数 `X` 强制转成 32-bit `int`：

```cpp
int q_low = context_low->parms().coeff_modulus().back().value();
int signed_X = (X > Q / 2) ? (int)X - Q : X;
int clean_m = (signed_X % q_low);
```

由于 `q_low` 约为 `2^30`，若要避免 `int` 溢出，需要：

$$
|k \cdot q_{low}| < 2^{31}
$$

也就是：

$$
|k| < 2
$$

因此只有当溢出多项式 `k(x)` 的系数几乎都在 `{-1, 0, 1}` 时，模拟才会正常工作。

### 私钥稀疏性

在 CKKS 中，溢出多项式主要来自 `c_1(x) * s(x) / q_low`。如果私钥 `s(x)` 是普通稠密多项式，卷积会让 `k(x)` 系数迅速变大，触发 `int` 溢出。题目反过来利用这个实现限制，暗示私钥必须极稀疏。

于是攻击不再是求解 RLWE，而是搜索低汉明重量私钥。官方脚本思路是枚举形如：

$$
s(x) = \pm x^a \pm x^b
$$

的二项式私钥，并用 `ciphertext_mask.dat` 校验候选是否能产生接近 mask 的解密结果。

### 求解步骤

求解程序先分别构造单项式私钥，记录每个 `±x^i` 对 `ciphertext_mask.dat` 的解密贡献：

```cpp
create_single_bit_key(sk1, n, i, val, context);
dec1.decrypt(c_zero, m);
mid_list.push_back({m, i, val});
```

然后枚举第二个单项式，与第一部分贡献相加，和零私钥解出的 `c0[0]` 做匹配：

```cpp
sum_val = (m_mid_val + p2_val) % modulus;
diff = min(abs(sum_val - c0_val), modulus - abs(sum_val - c0_val));
```

匹配到两个位置和符号后，构造最终私钥，解密 `ciphertext.dat`，再用 CKKS decoder 还原 flag 字符。

## 方法总结

- 核心技巧：利用 bootstrapping 模约减实现中的 32-bit 截断，反推出私钥必须低汉明重量，再枚举稀疏私钥。
- 识别信号：密码学实现中出现大模数 CRT 还原后强转小整数，且正确性依赖中间项不溢出。
- 复用要点：遇到 FHE/格密码题时，不要只从困难假设入手；实现层的数值范围、类型转换和近似误差经常会把理论问题降成小规模搜索问题。
