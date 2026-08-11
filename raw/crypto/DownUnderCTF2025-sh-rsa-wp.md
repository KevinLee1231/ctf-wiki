# SH-RSA

## 题目简述

服务生成 4096 bit RSA 模数 $N$，以 $e=65537$ 为公钥，并给出 92 个对消息 `00` 至 `5b` 的裸 RSA 签名：

$$
h_i=\operatorname{SHAKE128}(\operatorname{long\_to\_bytes}(N)\parallel m_i)[0:8],
\qquad s_i=h_i^d\bmod N.
$$

目标是给消息 `challenge` 伪造签名。验证函数并未检查 $s^e\bmod N$ 是否等于 64 bit 哈希整数，而是比较 `long_to_bytes(s^e mod N)[:8]` 是否等于该 8 字节哈希。也就是说，它只检查结果的高 8 字节前缀。

## 解题过程

### 利用裸 RSA 的乘法性

令目标前缀 $T=H(N,\texttt{challenge})$。对于任意签名子集 $I$，有

$$
s=\prod_{i\in I}s_i
\quad\Longrightarrow\quad
s^e\bmod N=\prod_{i\in I}h_i\bmod N.
$$

因此不需要私钥：只需找一个子集，使哈希乘积的最高 64 bit 正好是 $T$。成功候选还必须处在不发生模 $N$ 回绕的范围内；否则原始整数的前缀与取模结果无关。官方解法的有效候选满足这一隐含条件，实作时应显式检查 `product < modulus`。

另一个细节是 `long_to_bytes` 会丢掉前导零。严格说，若 $T<2^{56}$，目标的第一个字节为 `00` 而最短编码不足 8 字节，比较不可能成立。官方 `solv.py` 更保守地在 `T.bit_length() < 63` 时断开重试，以避免较短前缀带来的边界情形；这是一项搜索策略，并非验证函数的精确可达性判据。

### 四组 meet-in-the-middle 子集乘积

92 个 64 bit 哈希不能枚举全部 $2^{92}$ 个子集。官方 Rust solver 把它们切成 4 组、每组 23 个，再为每一组枚举 $2^{23}$ 个子集。枚举过程只保留乘积的高若干位：

```rust
fn msb_clamped(x: u128, bits: u32) -> u128 {
    let len = 128 - x.leading_zeros();
    if len <= bits { x } else { x >> (len - bits) }
}
```

先让第 1、2 组的候选乘积近似匹配一个中间目标，得到 `S12`；第 3、4 组同理得到 `S34`。最后对 `S12 * S34` 比较目标的高 56 bit，并对命中的 bitmap 用完整大整数重新相乘，检查：

1. 乘积的位长是 8 的倍数，使首字节边界与 `[:8]` 一致；
2. 十六进制表示以 $T$ 的十六进制表示开头；
3. 为保证验证语义，再确认完整乘积小于 RSA 模数 $N$。

这是近似乘积匹配：截断值只用于筛选，真正决定候选的是最后的完整乘积验证，不能只根据 64 bit 截断表直接发送签名。

### 组合签名并提交

Rust 程序输出选中的 bitmap 后，Python 包装层把对应的原始签名相乘并发送：

```python
signature = 1
product = 1
for i, (sig_i, hash_i) in enumerate(zip(padded_signatures, padded_hashes)):
    if bitmap & (1 << i):
        signature *= sig_i
        product *= hash_i

assert product < modulus
assert long_to_bytes(product)[:8] == target_hash
send_hex(signature)
```

其中 `padded_signatures` 与 `padded_hashes` 可按四段“23 个真实值 + 9 个乘法单位元 1”拼成 128 项；这是为了让 Rust 输出的四个 32 bit bitmap 与 Python 侧索引一致。官方包装器的签名数组实际用 0 填这 9 个位置，并依赖 Rust solver 永不选中 padding；这里改成 1 只为让验证片段在意外选中 padding 时仍有定义，对正常 bitmap 等价。

服务端的 `pow(signature, e, N)` 会自动做模约化；在 `product < modulus` 的候选上，结果就是所验证的原始乘积。官方 solver 用已编译的 Rust release 程序完成搜索，以适应 46 秒 alarm 的限制。

### 验证

官方 `solv.py` 收集 92 条签名，调用 Rust solver 找 bitmap，再把选中签名的乘积提交给 `s:`。题目配置给出的验证材料为 `DUCTF{how_did_you_break_64_bits_in_46_seconds!?}`。本归档没有运行这一高内存/并行子集搜索；公式与搜索分组均来自随附的 Python 和 Rust solver。

## 方法总结

- 核心技巧：裸 RSA 签名可相乘；一旦验证只检查模幂结果的短前缀，就能把伪造化成 hash 值的子集乘积前缀匹配。
- 识别信号：签名验证使用切片、字符串前缀或不固定长度的整数编码时，要先检查是否从“完整等值”退化成“仅比较高位”，并审计整数转字节的前导零语义。
- 复用要点：对子集乘积可按高位截断做多阶段 MITM，但最后必须用完整整数复核位长、前缀和模数回绕。近似表只是加速器，不是密码学验证。
