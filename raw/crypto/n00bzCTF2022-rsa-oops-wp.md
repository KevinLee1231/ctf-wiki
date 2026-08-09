# RSA-OOPS

## 题目简述

自制加密把 1024 位输出拆成 $X\|Y$：大模数分量为 $X=(m+r)^{17}\bmod N_L$，小模数分量为 $Y=X^{d_R}\bmod N_R\oplus r$。同一服务会用固定 `SERVER_SEED` 重建随机池，因此不同实例在 $N_R$ 位长相同时会复用同一组 $r$，只是顺序被打乱。

## 解题过程

公开密钥和密文都按 `KL=819`、`KR=205` 拆分：

```python
NL = public_key >> KR
NR = public_key & (2 ** KR - 1)
X = ciphertext >> KR
Y = ciphertext & (2 ** KR - 1)
```

$N_R$ 只有约 203–204 位，可以分解为 $p_Rq_R$ 并计算：

$$d_R=65537^{-1}\bmod (p_R-1)(q_R-1).$$

于是每条密文的随机量可直接恢复：

$$r=Y\oplus X^{d_R}\bmod N_R.$$

把一个实例的 128 条密文整理成 `r -> X` 字典，然后耗尽池并创建新实例。构造函数中的 `random.Random(SERVER_SEED)` 会为相同的 `NR.bit_length()` 生成同一组 128 个随机数；虽然随后用全局状态洗牌，但字典消除了顺序影响。

收集 17 个 $N_R$ 位长相同的实例。对任意共同的 $r$，其 17 个大分量满足：

$$X_i\equiv(m+r)^{17}\pmod{N_{L,i}}.$$

用 CRT 合并后，模数乘积足够大，可得到整数意义下的 $(m+r)^{17}$。不必直接开 17 次根；对多个随机量计算：

$$A_r=(m+r)^{17}-r^{17}.$$

展开式的每一项都含因子 $m$，因此数个 $A_r$ 的最大公约数会消去其余偶然因子并留下消息整数：

```python
values = []
for r in common_random_values[:5]:
    lifted = crt([table[r] for table in x_tables], NLs)
    values.append(int(lifted) - pow(r, 17))

m = values[0]
for value in values[1:]:
    m = gcd(m, value)
print(int(m).to_bytes(128, "big").lstrip(b"\x00"))
```

仓库 README 仍写着占位 flag，但实际 `flag.txt` 与官方 solver 可恢复的内容是：

```text
n00bz{turns_0ut_0r1g1n4l1ty_1snt_4lw4ys_0pt1m4l_s0_st1ck_t0_th3_st4nd4rds}
```

## 方法总结

本题同时利用了过小的辅助 RSA 模数、跨实例复用的随机池和无填充小指数广播。对齐随机量时必须按 $N_R$ 位长分组，并以恢复出的 $r$ 为键；直接按密文出现顺序配对会被洗牌破坏。
