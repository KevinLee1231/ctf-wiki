# spidr

## 题目简述

程序读取一个无符号 64 位十进制数 `e`，经 100 个高度混淆的 dispatcher 函数原地变换 `d`，仅当最终值为 `0x67696d65666c6167` 时打印 `grey{e}`。每个 dispatcher 把 32 位 `tmp` 与大量随机常量比较；命中分支后更新 `tmp`，并对 `d` 做加、异或或乘以奇数。题目的关键不是图片界面，而是从 compare-chain 中恢复唯一执行路径并逆转 64 位运算，归为 reverse。

生成器保证乘数为奇数，因此在模 $2^{64}$ 下存在乘法逆元；这使整个校验链可从最终常量倒推，无需在 $2^{64}$ 输入空间暴力。

## 解题过程

### 提取每个 dispatcher 的状态边

打开 `chal` 后，从入口 dispatcher 开始扫描指令。每个 `cmp tmp, imm` 给出一条边的当前状态；紧随其后的 `mov tmp, imm` 给出下一状态，附近的 `imul`、`add` 或 `xor` 及其立即数给出对 `d` 的操作。遇到 `call` 则进入下一个 dispatcher。

可以将每条命中的边抽象为：

```text
tmp == current  ->  tmp = next; d = op(d, constant)
```

仓库的 `sol/solve.py` 是可直接粘到 IDAPython 控制台的提取器。它从 `0x75C10` 开始收集每个函数最多 100 条状态边，再沿 call target 收集全部 100 个 dispatcher。

### 逆向算术链

从最终目标 $d=0x67696d65666c6167$ 与末尾 `tmp` 状态开始，依据 `next -> current` 反向查边；各运算的逆如下：

$$
\begin{aligned}
d'&=d+c &\Rightarrow&\ d=d'-c\pmod{2^{64}},\\
d'&=d\oplus c &\Rightarrow&\ d=d'\oplus c,\\
d'&=d\cdot c &\Rightarrow&\ d=d'\cdot c^{-1}\pmod{2^{64}}.
\end{aligned}
$$

最后一式可以使用 Python 的 `pow(c, -1, 2**64)`，其成立条件正是 $c$ 为奇数。对 100 个 dispatcher 依次做这些逆操作，得到原始十进制输入 `4022823573008984730`。

将这个数输入程序，终端条件成立并输出：

```text
grey{4022823573008984730}
```

## 方法总结

- 核心技巧：把随机 compare dispatcher 还原为状态转换图，再从已知终值逆走可逆的 64 位算术链。
- 识别信号：大量 `cmp`/`mov` 状态常量包围 `imul`、`add`、`xor`，且最终只比较一个 64 位常量时，应检查能否构造反向字典。
- 复用要点：模 $2^n$ 下只有奇数乘数可逆；提取分支时应同时记录状态迁移和算术操作，最后以真实二进制的成功输出验证，而不是只相信反向计算结果。
