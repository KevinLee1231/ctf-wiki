# SIDH

## 题目简述

服务端在 $E_0/GF(p^2)$ 上随机选择 $P_A,Q_A,s_A$，公开

$$
R_A=P_A+s_AQ_A,
$$

然后接收玩家给出的点 $R_B$，以 $\langle R_B\rangle$ 为核构造同源映射。玩家需要提交双方继续取商后所得曲线的 $j$-不变量。

这道题不需要利用针对 SIDH 的密码分析成果。服务端已经公开自己的核生成点 $R_A$，玩家只需按源码完成另一侧的同源交换，就能独立算出相同的共享 $j$-不变量。

## 解题过程

服务端收到 $R_B$ 后计算

$$
\phi_B:E_0\rightarrow E_B=E_0/\langle R_B\rangle,
$$

并利用同态性质把源码中的

$$
\phi_B(P_A)+s_A\phi_B(Q_A)
$$

化为 $\phi_B(R_A)$，最后得到

$$
E_B/\langle\phi_B(R_A)\rangle.
$$

玩家一侧以公开的 $R_A$ 构造

$$
\phi_A:E_0\rightarrow E_A=E_0/\langle R_A\rangle,
$$

再计算 $E_A/\langle\phi_A(R_B)\rangle$。两条路径对应依次对同一组核取商，所得曲线同构，因此 $j$-不变量相同。

下面的 SageMath 辅助脚本生成玩家点 $R_B$，并计算需要提交的共享值：

```sage
from random import randint


ea, eb = 110, 67
p = 2**ea * 3**eb - 1
F.<i> = GF(p**2, modulus=[1, 0, 1])
E0 = EllipticCurve(F, [1, 0])


def coefficients(value):
    # Sage 的列表顺序为 [常数项, i 的系数]；协议要求反过来发送。
    coeffs = list(value)
    coeffs += [0] * (2 - len(coeffs))
    return int(coeffs[1]), int(coeffs[0])


def encode_point(point):
    x, y = point.xy()
    xi, xc = coefficients(x)
    yi, yc = coefficients(y)
    return f"{xi},{xc},{yi},{yc}"


PB = E0.random_point()
QB = E0.random_point()
sB = randint(0, 3**eb)
RB = PB + sB * QB
assert not RB.is_zero()

print("发送给服务端的 RB：")
print(encode_point(RB))

raw = input("按 i系数,常数项,i系数,常数项 粘贴服务端 RA：\n")
rai, rac, rayi, rayc = [int(value) for value in raw.split(",")]
RA = E0(rai * i + rac, rayi * i + rayc)

phi_A = E0.isogeny(RA, algorithm="factored")
E_A = phi_A.codomain()
assert E_A.is_isogenous(E0)

shared_kernel = phi_A(RB)
phi_shared = E_A.isogeny(shared_kernel, algorithm="factored")
secret = phi_shared.codomain().j_invariant()

si, sc = coefficients(secret)
print("发送给服务端的 secret：")
print(f"{si},{sc}")
```

实际交互顺序如下：

1. 连接服务端，记录其输出的 `RA`；
2. 运行辅助脚本，把脚本输出的四个 `RB` 系数发送给服务端；
3. 将 `RA` 的两个坐标依次拆为 `i` 的系数和常数项，粘贴给辅助脚本；
4. 把脚本输出的两个 `secret` 系数发送给服务端，即可收到 `[+] flag:...`。

仓库中的 `SIDH` 目录未包含 `secret.py`，因此无法从这份源码快照核实固定 flag 字符串；应以比赛服务实际返回值为准。另一个离线文件 `task.sage` 在重组答案时误写成了 `secret[1]`，部署使用的 `main.py` 则正确使用 `secret_[1]`，复现应以后者为准。

## 方法总结

本题考查的是同源版 Diffie–Hellman 的交换流程，而不是“SIDH 已被攻破”这一外部结论。抓住同源映射的同态性质后，服务端的共享核可直接写成 $\phi_B(R_A)$；玩家选择并保存自己的 $R_B$，沿对称路径计算 $\phi_A(R_B)$，即可得到相同的共享曲线不变量。
