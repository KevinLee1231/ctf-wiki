# OhMyTetration

## 题目简述

服务把 flag 转成不超过 512 位的整数 $x$，随机选择一个大素数 $P$，并构造欧拉函数链

$$
P,\ \varphi(P),\ \varphi^2(P),\ \ldots,\ 1.
$$

选手可以指定与链上每个模数都互素的底数 $g$，再指定幂塔高度 `times`。服务返回的不是直接计算巨大整数后的取模，而是按欧拉链从内向外递推：

```python
def tetration(g, times, x, phi_chain):
    if times > len(phi_chain):
        return tetration(g, len(phi_chain), 0, phi_chain)
    res = x
    for i in range(times - 1, -1, -1):
        res = pow(g, res, phi_chain[i])
    return res
```

每次连接只能取得一个真正包含 flag 的票号，随后程序退出。另一个调试选项允许用自选的 $P,x,g$ 和高度测试同一算法，但不能直接读出真实 $x$。目标是把不同高度的票号当作 oracle，逐级恢复 $x$ 在欧拉链各层模数下的剩余类。

## 解题过程

### 1. 理解欧拉链为何泄露低层剩余类

当 $\gcd(g,m)=1$ 时，$g^e\bmod m$ 只依赖 $e\bmod\varphi(m)$。题目恰好在每一层使用下一项欧拉函数作为内层模数，因此高度为 $k$ 的结果只保留 $x$ 在某个较深链元素下的剩余信息。

当欧拉链已经到达 1 后，再增加幂塔高度不会增加新信息，结果会固定。反过来，从链尾向前减少高度，就能逐步观察更大的模数：已知

$$
x\equiv r\pmod {m_{i+s}}
$$

时，若 $m_{i+s}\mid m_i$，则所有候选为

$$
x=r+j,m_{i+s},\qquad 0\le j<\frac{m_i}{m_{i+s}}.
$$

把这些候选代入本地 `tetration`，与同高度的远端票号比较，即可选出正确的 $j$，得到 $x\bmod m_i$。

这里不应把“$g$ 在每层都是原根”当成未经验证的前提。原根条件能保证映射尽量单射，但实际求解只需对候选逐个计算并与 oracle 比较；若同一高度发生碰撞，应换高度、缩小步长或换底数。

### 2. 选择欧拉链合适的素数

服务会从多个大素数中随机选择 $P$。对每个候选预先计算完整欧拉链，检查：

- 链尾是否足够长，能从很小的模数开始恢复；
- 相隔若干层的模数是否整除，且商足够小，便于枚举；
- 所选 $g$ 在各层是否与模数互素，并能区分候选剩余类。

原解选择 $g=5$ 和下面这个 $P$：

```text
670144747631070976739015819027954827310379693667090873445520193836663869580245599076670148076473491050020123654751096623483807617465722698994356143777563707
```

由于每次连接的 $P$ 随机，实际交互时先读取菜单选项 1 给出的 $P$；不是目标值就断开重连。不要把比赛期间的临时 IP 写进脚本，保留“连接服务并读取 $P$”这一接口即可。

### 3. 从链尾逐级提升模数

本地辅助函数如下：

```python
from sympy import totient

def get_phi_chain(P):
    chain = []
    while P != 1:
        chain.append(int(P))
        P = int(totient(P))
    return chain

def tetration(g, times, x, chain):
    if times > len(chain):
        return tetration(g, len(chain), 0, chain)
    res = x
    for i in range(times - 1, -1, -1):
        res = pow(g, res, chain[i])
    return res
```

`query(P, g, times)` 表示反复建立连接，直到服务选中目标 $P$，再提交 $g$ 和高度并返回票号。恢复框架为：

```python
g = 5
chain = get_phi_chain(P)

# 原题参数下，这一层只区分 x mod 2。
idx = len(chain) - 7
ticket = query(P, g, idx)
res = next(v for v in range(2) if tetration(g, idx, v, chain) == ticket)
modulus = 2

# 一次向外推进 5 层；每一步只枚举相邻已知模数之间的商。
step = 5
for i in range(idx - step, 1, -step):
    assert chain[i] % chain[i + step] == 0
    ticket = query(P, g, i)
    quotient = chain[i] // chain[i + step]

    for j in range(quotient):
        candidate = res + j * modulus
        if tetration(g, i, candidate, chain) == ticket:
            res = candidate
            modulus = chain[i]
            break
    else:
        raise ValueError("no residue matched; change the step or height")
```

循环结束时已经得到 $x\bmod m$，其中 $m$ 接近 $P$ 但还不一定覆盖全部 512 位。最后查询高度 1；此时票号就是 $g^x\bmod P$，只需枚举剩余的少量同余类：

```python
ticket = query(P, g, 1)
for k in range((P + modulus - 1) // modulus):
    candidate = res + k * modulus
    if pow(g, candidate, P) == ticket:
        x = candidate
        break
else:
    raise ValueError("final residue not found")
```

最后使用 `long_to_bytes(x)` 恢复 flag，并用题目要求的前缀和闭合花括号验证结果。

## 方法总结

- 核心技巧：把不同高度的模幂塔当作欧拉函数链上的剩余类 oracle，从链尾的小模数开始逐级提升。
- 识别信号：程序显式递归计算 $\varphi(P)$、允许选择幂塔高度，并把秘密放在最内层指数时，应检查每个高度实际泄露的是秘密模哪个链元素。
- 复用要点：先筛选链结构和底数，再枚举相邻模数之商；所有候选都用本地同算法和远端票号验证，不依赖“每层均为原根”的理想化假设。
