# MiniLCTF 2021 - tukuai（土块）

## 题目简述

服务端每轮随机生成两个经典比特 `coin1`、`coin2`，据此初始化一个两量子比特态，然后测量量子比特 1 并公开结果；玩家必须猜出 `coin2`。用户名为 `lubenwei` 时，可以在游戏开始前提交一串量子门作为“作弊电路”。连续获胜约 42 轮、把资金从 20 万增加到 500 万以上即可取得 flag。

核心不是预测随机数，而是设计一个可逆量子线路，让最终被测量的量子比特 1 恰好等于未公开的 `coin2`。

## 解题过程

初始化代码使用

```python
temp = coin1 * 2 + coin2
init_state[temp] = 1
```

Qiskit 的两比特基态索引按 $|q_1q_0\rangle$ 解释，因此初态为

$$
q_1=coin1,\qquad q_0=coin2.
$$

依次施加两次 CNOT：

1. `cx(1, 0)`：$q_0\leftarrow coin2\oplus coin1$；
2. `cx(0, 1)`：$q_1\leftarrow coin1\oplus(coin2\oplus coin1)=coin2$。

服务端最后执行 `measure(1, 0)`，所以公开的 `servercoin` 就是本轮答案。操作码 9 对应 CNOT，作弊序列应为：

```text
username: lubenwei
op: 9
params: 1 0
op: 9
params: 0 1
op: 0
```

自动交互脚本如下：

```python
from pwn import remote

p = remote("127.0.0.1", 10000)
p.sendlineafter(b"what's your name", b"lubenwei")
p.sendlineafter(b"op:", b"9")
p.sendlineafter(b"params:", b"1 0")
p.sendlineafter(b"op:", b"9")
p.sendlineafter(b"params:", b"0 1")
p.sendlineafter(b"op:", b"0")

for _ in range(42):
    p.recvuntil(b"my coin is ")
    answer = p.recvuntil(b" your coin is?", drop=True)
    p.sendline(answer)

print(p.recvall().decode(errors="replace"))
```

每次答对增加 114514，42 次后资金超过 500 万，下一轮循环开头会输出 flag。远程 flag 为运行环境变量，仓库中未保存固定值。

## 方法总结

关键是先确认 Qiskit 的比特序与测量对象。两个 CNOT 实际实现了一个线性变换，把原来的 $q_0$ 信息搬到 $q_1$；用代数逐步写出每个比特，比凭电路图猜测更稳妥。交互脚本应直接回送服务端测量结果，无需模拟量子态，也不应把动态 flag 写死在 WP 中。
