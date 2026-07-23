# no-brainrot-allowed

## 题目简述

服务端使用 1024 位 RSA 和 $e=65537$ 加密 flag，并提供一个受限解密接口。接口不返回明文，只泄露解密结果的十六进制表示是否以 `67` 开头。这个响应会对应多个不同数量级的区间；官方脚本再把同一密文乘以 $16^e$ 复查，从而把它收紧为最高的 1024 位目标区间。

RSA 的乘法同态允许把目标密文乘以任意 $s^e$，从而询问 $m s\bmod n$ 是否落入指定区间。官方解法据此执行类似 Bleichenbacher 的区间收缩攻击。

## 解题过程

令模数长度为 1024 位，并设

$$
B=2^{1016},\qquad
L=0x67\cdot B,\qquad
U=0x68\cdot B-1.
$$

解密值的首字节为 `0x67` 等价于 $L\leq m\leq U$。对目标密文 $c=m^e\bmod n$ 构造

$$
c'=c\cdot s^e\bmod n,
$$

则解密结果为 $m'=ms\bmod n$。接口响应由此告诉我们是否存在整数 $r$，使

$$
L\leq ms-rn\leq U.
$$

若当前候选明文区间为 $[a,b]$，对某个命中的 $s$ 和可能的 $r$，新区间为

$$
\left[
\max\left(a,\left\lceil\frac{L+rn}{s}\right\rceil\right),
\min\left(b,\left\lfloor\frac{U+rn}{s}\right\rfloor\right)
\right].
$$

攻击流程如下：

1. 随机寻找一个乘数 $s_0$，使盲化后的明文落入 `0x67` 区间；
2. 初始化候选区间集合，并按 oracle 结果寻找下一组有效 $s$；
3. 枚举与每个旧区间相交的 $r$，用上式生成新区间；
4. 合并相交区间，重复直到集合只剩一个整数；
5. 乘以 $s_0^{-1}\bmod n$ 取消最初的盲化。

服务端仅判断十六进制字符串前缀，所以较短的数也可能以 `67` 开头。官方复合 oracle 只有在原查询命中 `67`、而乘以 $16^e$ 后不再命中时才返回真；这利用高位区间乘 16 后发生模 $n$ 回绕的特征，排除了低数量级的同前缀区间。最终整数转为大端字节串即可恢复：

```text
UMDCTF{thank_you_for_helping_the_Counter-Brainrot_Terrorism_Unit_(or_CBT_Unit_for_short)_with_their_operations}
```

## 方法总结

只泄露一个前缀判断并不代表 RSA 解密接口安全。教科书 RSA 的乘法同态把这一位信息放大成可重复的区间查询，攻击者能够逐轮缩小明文范围。实际系统应使用经过证明的填充方案，并彻底移除可区分的解密错误。
