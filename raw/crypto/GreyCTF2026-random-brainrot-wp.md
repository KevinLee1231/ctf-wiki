# random_brainrot

## 题目简述

服务用 logistic map 生成字节流，并与一个固定 16 字节 `SECRET` 循环异或。客户端可以反复提交任意明文并得到密文；若提交的明文本身等于 `SECRET`，服务直接返回 flag。

状态更新和输出为：

$$
x_{n+1}=r x_n(1-x_n),\qquad b_n=\lfloor256x_n\rfloor
$$

其中 $r$ 会在连接建立时公开，范围约为 $[3.676,3.676767676767)$。这一混沌映射的量化输出并非均匀随机字节。虽然初始 $x$ 未知，长期输出的偏置分布仍可被统计建模。

## 解题过程

提交 16 字节全零明文时，返回值满足：

$$
c_{j,t}=b_{j,t}\oplus secret_j
$$

同一密钥位置收集足够多的查询结果后，可以对每个 `secret_j` 独立做最大似然估计。服务开头已经打印精确到 15 位小数的 $r$，因此在本地随机选取大量初态，执行相同的 1000 次预热，再采样量化字节，估计分布 $P_r(b)$：

```python
counter = collections.Counter()
for _ in range(5000):
    simulator = PRNG(r, random.random())
    for value in first_16_bytes(simulator):
        counter[value] += 1

cost[b] = -math.log(max(counter[b], 1) / total)
```

对远端连续提交约 100 次 `00` 重复 16 字节的明文，并按位置保存密文字节。枚举某一位置的 256 个候选密钥 $g$，将观测反异或为 $c\oplus g$。若 $g$ 正确，这些值应服从模拟得到的 logistic-map 输出分布；若 $g$ 错误，分布会被 XOR 错位。负对数似然为：

$$
\operatorname{score}_j(g)=-\sum_t\log P_r(c_{j,t}\oplus g)
$$

选择分数最小的候选：

```python
for pos in range(16):
    best_key = min(
        range(256),
        key=lambda guess: sum(cost[c ^ guess] for c in samples[pos]),
    )
    secret[pos] = best_key
```

恢复 16 个位置后，把 `secret.hex()` 作为下一条输入发送。服务在加密前检查 `msg == SECRET`，因此会进入隐藏分支并输出：

```text
grey{d4t4_4n4ly71c5_15_m4th_15_cRyp70_l0l_s1xs3v3n67676767}
```

若一次恢复有少数字节出错，可以重新连接并增加本地建模次数或远端样本数；每个连接会公布自身的 $r$，必须为该连接重新建立分布模型。

## 方法总结

混沌不等于密码学安全。这里无需恢复连续状态 $x$，也无需预测精确的下一字节；固定密钥重复使用，使每个密钥字节都能通过输出分布偏差独立估计。选择明文为全零把问题化为“哪个 XOR 平移最符合已知分布”，最大似然评分则把大量弱偏差累积成可靠区分信号。
