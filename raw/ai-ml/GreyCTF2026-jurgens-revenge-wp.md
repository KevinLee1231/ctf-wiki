# Jurgen's Revenge

## 题目简述

本题的公开检查器只接受形如 `grey{...}` 的固定长度、由小写字母数字和下划线组成的候选，并只输出 `accepted` 或 `rejected`。但附件并非纯远程黑盒：它提供 `model.pt`、字母表和完整的模型运行时。该运行时是一个固定参数的字符级循环验证器，表面结构是 embedding、循环核心、线性 readout 与分类头。

每一轮的状态并不直接暴露，而是经由随轮数变化的正交矩阵混合。模型将 cell 状态逆变换成 packed chart，计算硬阈值门和累加记忆，再乘回下一轮的正交矩阵。最终分类头只接受某个 $55$ 字符路径。主要障碍是从公开神经网络参数中恢复有限状态约束与终态条件，故归入 `ai-ml`。

## 解题过程

### 消去每轮状态混合

模型用固定 seed 生成每轮正交矩阵 `self._cw[step]`，并保存其转置 `self._cu[step]`。由于正交矩阵的逆就是转置，任意 cell 都可恢复到稳定坐标系：

```python
def decode(step, cell):
    return model._cu[step] @ cell.to(dtype=torch.float64)

def encode(step, packed):
    return model._cw[step] @ packed.to(dtype=torch.float64)
```

在 packed chart 中，前半部分是取值为 $-1$ 或 $1$ 的门状态，后半部分是按字符累加的 memory。真实一步转移等价于：

$$
\begin{aligned}
g_t &= \operatorname{sign}(W^{(t)}_e e(x_t)+W^{(t)}_c r_t+b_t),\\
m_{t+1} &= m_t+128\,V^{(t)}[x_t].
\end{aligned}
$$

这里 $r_t$ 是 packed state 的 readout。先用一个已知探针串跑完整个公开模型，比较每个二值坐标随字符的移位模式，即可识别它们中哪些坐标保存上一字符和上两字符的 one-hot 寄存器。轮号由循环下标提供，因此不需要另外猜测 phase register。

### 从终态因果特征恢复转移关系

分类器先对 `[readout, packed]` 产生 evidence 位，再把 evidence 加权成接受/拒绝。逐个翻转终态 packed 的二值位，并观察能否改变正权重 evidence 行，可以筛出真正影响接受的因果特征；上一字符/上两字符的 bookkeeping 位从候选中剔除。

然后构造人工 packed state：其他二值位固定为 $1$，只将已经恢复的 `prev1`、`prev2` 寄存器设置为指定字符。把它重新编码成 cell，喂给真实的 `_advance`，再 decode 输出，即可直接测量某个因果位在给定 `(step, prev, current)` 下是否保留：

```python
cell = encode(step, patched_packed(step, prev2, prev1))
next_cell = model._advance(step, cell, current)
survives = decode(step + 1, next_cell)[causal_dim] > 0
```

对每个因果位记录首字符允许集合和其后每一位置的允许二元组 `(previous, current)`。这把看似不透明的 RNN 转为若干位置相关的正规语言约束。单一约束会残留诱饵路径，因此需要把多个因果位的允许集合逐位置取交集。

### 加入记忆条件并用 Z3 证明唯一性

终态 evidence 的线性系数还泄露了哪些 memory 坐标被使用。对于同一 memory 坐标，若多条 evidence 不等式给出一致边界，就把其四舍五入结果作为终态目标。字符 $x_t$ 的累加贡献可直接从公开的 `core.value.weight` 取出：

$$
\sum_{t=0}^{54}128\,V^{(t)}[x_t,j]=M_j.
$$

令 `x_t` 为字母表下标。Z3 对其施加字符范围、首字符、相邻转移和上述记忆等式，先求出一个可满足串；随后加入“至少一个位置不同”的析取条件并再次求解。第二次为 `unsat` 即证明该恢复模型下的候选唯一。

```python
solver.add(Or([x[0] == first for first in start]))
for step, pairs in enumerate(allowed_pairs, start=1):
    solver.add(Or([And(x[step - 1] == a, x[step] == b) for a, b in pairs]))
solver.add(Or([x[i] != solution[i] for i in range(length)]))
```

最后仍通过原始 `model.run_payload()` 检查候选，避免把只满足某个诱饵电路的字符串当作答案。运行官方求解器：

```bash
python solve/solve.py --report
```

输出会报告 `Z3 uniqueness: proved`，并恢复 `grey{h1y4_there_n3el_n4nda_d1dnt_s3e_y0u_0ver_fr0m_ov3r_h3re}`。

## 方法总结

- 核心技巧：可逆线性状态混合不是加密边界；将其反变换后，可把循环神经验证器还原为有限状态转移和线性记忆约束。
- 识别信号：公开 `state_dict` 同时含有循环运行时代码、固定 transport 矩阵和二值门时，应尝试 activation chart、因果翻转与状态 patching，而不应只进行随机输入查询。
- 复用要点：先从已知探针恢复寄存器语义，再只保留对终态有因果影响的位。将恢复出的局部关系交给 SMT，并把候选回送真实模型，是防止 decoy constraint 的关键闭环。
