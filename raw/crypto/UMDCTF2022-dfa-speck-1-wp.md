# DFA Speck 1

## 题目简述

题目针对 Speck 128/128，在最后一轮输入的 64 个比特位置逐一注入精确单比特故障，并给出正确密文与故障密文。目标是恢复第 32 轮使用的 64 位轮密钥 `k32`。

Speck 的 ARX 结构让单比特差分是否触发加法进位直接泄露密钥位；最高位存在一个无法由公式区分的二义性，题目额外提示正确结果按 ASCII 可读。

## 解题过程

Speck 128/128 的轮函数可写为：

$$
x'=(\operatorname{ROR}_8(x)+y)\oplus k,\qquad
y'=\operatorname{ROL}_3(y)\oplus x'.
$$

对正确输出 $(x_t,y_t)$ 和故障输出 $(x'_t,y'_t)$，先计算：

$$
e=\operatorname{ROR}_3(x_t\oplus y_t\oplus x'_t\oplus y'_t).
$$

再观察 $x_t\oplus x'_t$。若该差分只有一个置位，比特翻转没有在模加法中继续传播；若出现连续传播，则说明该位置产生了进位。按最低位到最高位维护进位状态 $c_i$，即可逐位恢复轮密钥：

```python
kt = 0
carry = 0

for i in range(64):
    # ct 与 faulty 是正确/故障末轮输出
    propagation = popcount(x_t ^ x_faulty)
    old_carry = carry

    if propagation == 1:
        add_bit = carry
        carry = 0
    else:
        add_bit = carry ^ 1
        carry = 1

    key_bit = (
        add_bit
        ^ bit(x_t ^ y_t, (i + 3) % 64)
        ^ bit(x_t, i)
        ^ old_carry
    )
    kt |= key_bit << i
```

最高位无法单靠这组差分确定。对两个候选分别转成大端字节，只有把最高位置 1 后得到可读 ASCII：

```text
k32 = 306e675f336e7566
ASCII = 0ng_3nuf
```

按题目要求提交十六进制轮密钥：

```text
UMDCTF{306e675f336e7566}
```

## 方法总结

- ARX 密码的故障差分会暴露加法进位链；判断差分传播长度即可递推密钥位。
- 正确密文和每个比特位置的故障密文必须按相同的 64 位字序、端序解析，不能把文件中的拼接顺序直接当主机端整数顺序。
- 公式无法恢复的最高位应保留两个候选，再用题面给出的 ASCII 约束消除二义性。
- 第一问只要求末轮轮密钥，不需要逆完整 key schedule。
