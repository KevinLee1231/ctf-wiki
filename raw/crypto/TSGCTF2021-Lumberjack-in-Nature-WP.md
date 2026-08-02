# TSGCTF2021 Lumberjack in Nature WP

## 题目简述

附件给出两个大整数 `s`、`e`，并提供一个看似简单却不可直接运行的解码器：

```python
mp.dps = 1000000000

def decode(enc):
    return int(power(2, enc * ln(2)))

flag = decode(s << e)
print(flag.to_bytes((flag.bit_length() + 7) // 8, "big")[:74])
```

这里 $e=13371337$，指数 $s2^e\ln2$ 极大；按题面设置十亿位精度计算完整整数既浪费内存，也完全没有必要。输出只取整数的大端最高 74 字节，因此真正需要的是指数的小数部分，也就是巨数的有效尾数，而不是它后面数百万字节的零位移。

## 解题过程

令

$$
x=s2^e,\qquad y=x\ln2=q+\theta,
$$

其中 $q=\lfloor y\rfloor$、$0\leq\theta<1$。那么

$$
2^y=2^q\cdot2^\theta.
$$

$2^q$ 只负责把二进制小数点向右移动，整数最高位附近的内容由 $2^\theta$ 决定。flag 长 74 字节，首字符 `T` 的最高有效位位于整段的第 590 位，所以只需高精度求出 $\theta$，再计算

$$
\left\lfloor 2^\theta\cdot2^{74\times8-2}\right\rfloor.
$$

困难转化为求 $\{s2^e\ln2\}$。利用级数

$$
\ln2=\sum_{k=1}^{\infty}\frac{1}{k2^k},
$$

可写成

$$
\{2^e\ln2\}
=\left\{
\sum_{k=1}^{e}\frac{2^{e-k}\bmod k}{k}
+\sum_{k=e+1}^{\infty}\frac{2^{e-k}}{k}
\right\}.
$$

前一部分不需要生成 $2^{e-k}$ 本身，只要用快速模幂计算 `pow(2, e-k, k)`；后一部分按负指数直接累加，很快就小于所需精度。官方实现用 Rust、`rug::Integer` 和 Rayon 分块并行计算 $\{2^e\ln2\}$ 的 6000 个二进制位，核心等价于：

```rust
for k in 1..=e {
    let numerator = pow_mod(2, e - k, k);
    acc += (numerator << precision) / k;
    acc %= 1 << precision;
}

for k in (e + 1).. {
    acc += (1 << (e - k + precision)) / k;
    if acc no longer changes { break; }
}
```

设输出的 6000 位整数为 `f`，则它代表

$$
f/2^{6000}\approx\{2^e\ln2\}.
$$

乘上 `s` 后仍只保留小数部分：

```python
fraction_bits = int(binary_digits, 2)
precision = len(binary_digits)
theta_bits = (s * fraction_bits) % (1 << precision)
theta = mpf(theta_bits) / mpf(1 << precision)
```

最后恢复最高 74 字节：

```python
mantissa = int(
    exp(theta * ln(2)) * mpf(1 << (74 * 8 - 2))
)
print(mantissa.to_bytes((mantissa.bit_length() + 7) // 8, "big"))
```

得到：

```text
TSGCTF{sH4miKo,_muScl3_WIll_HeLp_You_wI7h_thE_pro6LeM._7r4in_YOUr_muscle.}
```

另一份官方求解器直接按同一 $\ln2$ 级数累加，并从指数中减去 8 的倍数，把巨大的二进制位移改成整字节位移；数学原理与上述方法相同，只是面对 $e=13371337$ 时，Rust 分块版本更实用。

## 方法总结

本题的核心是“只求巨数的最高有效字节”。把指数拆成整数部分和小数部分后，整数部分仅控制整体位移，小数部分决定最高位尾数；再用 $\ln2$ 的可逐位级数和模幂直接计算远处的二进制小数位，就能绕开十亿位浮点运算。遇到 `int(exp(巨大数))` 后只截取开头的题目，应优先推导 mantissa、bit length 和字节对齐关系，而不是照题目代码提高全局精度。
