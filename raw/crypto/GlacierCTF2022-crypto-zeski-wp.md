# GlacierCTF2022 - crypto_zeski

## 题目简述

程序用 Python `random.getrandbits(512)` 连续产生 $x,y$，再迭代仿射同余式 $z\leftarrow az+b\pmod m$ 共 $y$ 次，把结果与明文异或。附件给出 $a,b,m$、20 组已知明文对应的密文，以及每组的 `max(x, y)`；flag 使用紧随其后的两次 PRNG 输出加密。

## 解题过程

参数最关键的异常是：

$$
m=a\cdot b.
$$

一般的 $y$ 次迭代包含 $b(1+a+\cdots+a^{y-1})$，但模 $ab$ 时，除常数项 $b$ 外的每一项都含因子 $ab$。因此题目中的快速实现等价于：

$$
r=\operatorname{roll}(x,y)=a^y x+b\pmod m.
$$

对一组已知明文，先算 `r = ciphertext XOR bytes_to_long(plaintext)`，再模 $b$，得到 $r\equiv a^y x\pmod b$。服务泄漏的 `seed=max(x,y)` 不说明它是哪一个随机数，所以分别尝试两种方向：

```sage
def recover_y(r, x):
    return int((Mod(r, b) / x).log(Mod(a, b)))

def recover_x(r, y):
    return int(Mod(r, b) / pow(a, y, b))
```

若 `seed` 是 $x$，第一式通过离散对数恢复 $y$；若 `seed` 是 $y$，第二式直接恢复 $x$。只保留不超过 512 位且重新代入 `roll` 后完全匹配的候选。对少数仍有两个方向的样本，枚举这些很小的歧义组合即可。

Python 的 `random` 是 MT19937。一次 512 位 `getrandbits` 消耗 16 个 32 位输出；20 组 $(x,y)$ 一共给出 640 个连续输出，超过恢复完整状态所需的 624 个。按真实调用顺序把每个候选值交给 MT19937 predictor，恢复状态后预测 flag 使用的下一对 $x,y$：

```python
x = predictor.getrandbits(512)
y = predictor.getrandbits(512)
plain = long_to_bytes(flag_ct ^ roll_faster(x, y))
```

匹配 `glacierctf` 前缀的分支输出：

```text
glacierctf{Th0s3_p4r4m3t3r5_d0nt_l00k_r1ght}
```

## 方法总结

这条链同时利用了代数参数退化和非密码学 PRNG。$m=ab$ 把长迭代压缩成可逆的模 $b$ 关系，`max(x,y)` 又足以消除大部分顺序歧义；恢复 624 个 MT19937 状态字后，后续输出便不再保密。密钥材料应来自 CSPRNG，公开参数也必须检查是否引入意外因子关系。
