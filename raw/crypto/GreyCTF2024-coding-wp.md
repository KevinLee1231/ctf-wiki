# Coding

## 题目简述

服务端用 $100\times200$ 的二元生成矩阵编码 100 位消息，并以 $0.05$ 的独立概率翻转每个码字位。玩家可先指定生成矩阵 $G$，但服务端会返回经随机可逆矩阵 $S$ 和列置换 $P$ 隐藏后的 $M=SGP$。挑战共 200 轮，每轮可提交 20 个候选消息，至少答对 120 轮才能取得 flag。

## 解题过程

选择一个稀疏校验矩阵 $H$：取 100 行、200 列，每行恰有 6 个一，其余为零，再由它构造 LDPC 生成矩阵 $G$。左乘 $S$ 只改变生成空间的基，右乘 $P$ 只打乱坐标，所以服务端公开的 $M$ 仍生成同一个置换后的低密度码。

直接从 $M$ 求出的校验矩阵通常不再稀疏。官方解法先建立 `LinearCode(M)`，把校验基提升到整数环做 LLL，再降回 $\operatorname{GF}(2)$，恢复短而稀疏的校验向量：

```python
C = LinearCode(Matrix(GF(2), M))
H = (C.parity_check_matrix()
       .change_ring(ZZ)
       .LLL()
       .change_ring(GF(2))
       .numpy(dtype=int))
```

得到 $H$ 后，用已知信道错误率 $0.05$ 初始化 belief propagation 解码器。BP 输出偶尔还残留少量错误，因此预先枚举重量为 0、1、2 的错误向量 $e$，按综合 $He^T$ 建表：

```python
syndrome_errors = {}
for error in errors_of_weight_at_most_two(200):
    syndrome = tuple(H @ error.transpose() % 2)
    syndrome_errors.setdefault(syndrome, []).append(error)
```

对每轮密文 $c$，先取得 BP 估计码字 $w$，再计算 $Hw^T$。查表得到可能的低重量修正量后，逐个测试 $w+e$，并解线性方程

$$
mM=w+e
$$

恢复候选消息。每轮最多发送 20 个可解候选，正好利用题目提供的容错接口。完成不少于 120 轮后得到：

```text
grey{what_linear_code_did_you_use?_zcUQenJEv4wXNRYxB5pPkH}
```

## 方法总结

随机换基不会隐藏线性码本身，列置换也不会破坏低密度校验关系的重量。这里的关键是主动选择带有可恢复稀疏结构的码，再用 LLL 找回短校验向量；BP 负责大部分纠错，低重量综合枚举和每轮 20 次猜测负责吸收尾部误差。遇到被 $S$、$P$ 隐藏的线性码时，应区分“基被隐藏”和“码空间被改变”。
