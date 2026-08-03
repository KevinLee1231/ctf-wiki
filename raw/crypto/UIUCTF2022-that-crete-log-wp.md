# That-crete Log

## 题目简述

服务先生成随机 `token`，把 flag 整数与它异或得到指数

$$
m=\operatorname{bytes\_to\_long}(\text{flag})\oplus\text{token}.
$$

随后允许玩家五次提交“$\varphi(N)$ 的素因子”。程序令 $N=1+\prod f_i$，检查各个 $f_i$ 和 $N$ 看似为素数，再随机选择 $x$ 并返回 $x^m\bmod N$。如果能构造群阶完全已知且平滑的素数 $N$，就能求离散对数得到 $m$，最后与公开 token 异或恢复 flag。

限制本意是要求最大因子至少 512 位，从而阻止平滑阶离散对数；漏洞在于自制素性测试只做小素数试除，并对固定 Miller–Rabin 底数集合测试。可以构造对这些固定底数均为强伪素数的合数，把许多小因子伪装成一个“大素因子”。

## 解题过程

### 绕过“安全因子”检查

验证器使用的底数固定为：

```python
[2, 3, 5, 7, 11, 13, 17, 19, 31337]
```

并且只试除小于 256 的整数。因而可以预先寻找一个合数 $F$，满足：

- $F$ 没有小于 256 的因子；
- $F$ 对上述全部底数通过 Miller–Rabin；
- 已知 $F$ 的完整小素因子分解；
- 把 $F$ 与必要的少量小因子相乘后，$N=F\cdot s+1$ 是 512 到 1024 位的真素数。

提交时把 $F$ 作为单个“素数”交给服务，服务会接受它；但攻击者知道 $N-1$ 的真实分解，所以群 $\mathbb{F}_N^*$ 的阶对攻击者仍然是平滑的。官方仓库提供了五组预先生成并验证过的 $N$，避免在 60 秒交互限制内现场搜索。

### 对每轮求离散对数

每轮收到

$$
y=x^m\pmod N.
$$

因为 $N-1$ 的因子全部已知，SageMath 可以用 Pohlig–Hellman 求出 $m$ 在 $x$ 的阶上的剩余。官方脚本进一步按 $N-1$ 中的大素因子收集同余，以排除小子群不足以覆盖完整指数空间的问题：

```python
F = Zmod(N)
x = F(received_x)
y = F(received_y)
dlog = y.log(x)

for prime, multiplicity in factor(N - 1):
    if prime < 256:
        continue
    residues.append(int(dlog % prime))
    moduli.append(int(prime))
```

服务禁止重复使用同一个 $N$，所以使用五个不同的预计算素数。只要收集到的互素模数乘积大于 $m$ 的可能范围，就能通过中国剩余定理唯一恢复：

$$
m=\operatorname{CRT}(m_i\bmod r_i).
$$

最后撤销会话掩码：

```python
masked_flag = crt(residues, moduli)
flag_integer = int(masked_flag) ^ token
print(long_to_bytes(flag_integer))
```

官方参数恢复出的结果为：

```text
uiuctf{w0w_1_th0ughT_Th4t_discr3te_L0g_w4s_h4rD_f04_s4f3_prim2s!1!_d91ea3cf4a3daaf0604520}
```

## 方法总结

- 核心技巧：针对固定底数 Miller–Rabin 构造强伪素数，让服务误判 $N-1$ 含有超大素因子，同时保留攻击者已知的平滑分解，再用 Pohlig–Hellman 和 CRT 恢复指数。
- 识别信号：服务让用户提交因子、素性测试底数固定且数量少、群阶由用户输入直接决定，并返回形如 $x^m\bmod N$ 的 oracle。
- 复用要点：通过 Miller–Rabin 不等于已证明为素数；构造参数时既要满足服务端的伪素数条件，也要确保最终 $N$ 是真素数。离散对数只在基点实际阶上确定，必须收集足够同余并检查 CRT 模数乘积是否覆盖目标指数范围。
