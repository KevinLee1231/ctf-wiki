# janken vs yoshiking 2

## 题目简述

`crypto/janken-vs-yoshiking-2/challenge/server.sage` 与 `distfiles/server.sage` 的服务流程是：

- 服务器在域 $\mathbb{F}_p$ 中给出随机 $5\times 5$ 矩阵 $M$；
- 每一轮随机挑选 $y\in\{1,2,3\}$，并返回承诺矩阵 $C=M^r$，其随机数满足：
$$r\bmod 3 + 1 = y$$
- 玩家回传手势后判定胜负；胜利 100 次即得 flag。

表面上这是石头剪刀布交互，决定性机制却是矩阵幂承诺在行列式同态下泄露指数。只要标量底数的乘法阶含因子 3，就能稳定恢复 $r\bmod 3$，因此归入 `crypto`。

## 解题过程

`solution/solve.sage` 与自动导出的 `solution/solve.sage.py` 都给出直接复现步骤：

1. 与服务连接并读取第一行 `p` 与 `M`；
2. 每轮读取 `commitment is=[...]`，将其转成 `5x5` 矩阵 `C`；
3. 用行列式降维：
$$\det(C)=\det(M^r)=\det(M)^r$$
在 $\mathbb{F}_p^*$ 上求离散对数：
$$x=\mathrm{discrete\_log}(\det(C),\det(M))$$
若 $o=\operatorname{ord}(\det(M))$ 且 $3\mid o$，则 $x\equiv r\pmod o$ 的所有解在模 3 意义下相同，所以可以得到 $r\bmod 3$。固定素数满足 $3\mid(p-1)$；随机行列式的阶仍应显式检查。若 $3\nmid o$，本次投影无法区分三种手势，应重新连接取得新的随机矩阵，而不是继续猜。

4. 服务器手势为 $y=r\bmod3+1$，客户端还必须发送**克制它**的手势：

| $r\bmod3$ | Yoshiking | 获胜输入 |
|---:|---|---:|
| 0 | 1（Rock） | 3（Paper） |
| 1 | 2（Scissors） | 1（Rock） |
| 2 | 3（Paper） | 2（Scissors） |

5. 按最后一列连续发送 100 轮即触发通关输出。

```python
"""核心一行（来自 solution/solve.sage）"""
o = M.det().multiplicative_order()
assert o % 3 == 0
md = M.det()
cd = C.det()
x = discrete_log(cd, md)
yoshiking_hand = x % 3
hand = [3, 1, 2][yoshiking_hand]
sock.sendlineafter("hand(1-3): ", str(hand))
```

成功完成 100 轮后得到：

```text
CakeCTF{though_yoshiking_may_die_janken_will_never_perish}
```

## 方法总结

- 核心技巧：把矩阵指数承诺先映射到行列式标量，再做离散对数和模 3 解码；避免对 5x5 幂运算做复杂逆运算。
- 识别信号：只要题面出现 `M^r` 与公开 `M,p`，先看可否通过不变量（行列式、特征值）降维。
- 复用要点：同态降维后必须检查像元素的阶是否保留目标模数信息；只有 $3\mid\operatorname{ord}(\det(M))$ 时，离散对数代表元的模 3 才不受多解影响。
