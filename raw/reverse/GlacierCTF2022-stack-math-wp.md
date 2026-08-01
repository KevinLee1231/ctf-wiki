# GlacierCTF2022 - Stack Math

## 题目简述

服务随机给出一个 32 位素数，要求在 5 秒内提交少于 85 条指令的十六进制程序，使 Verilog 实现的 32 位栈机计算出该数。可用操作只有压入 0/1、自增自减、交换、复制、加减、乘法和结束。

## 解题过程

每个十六进制半字节的高 3 位选择指令，低 1 位是修饰位。按实际 Verilog，关键编码为 `0=ZERO`、`1=ONE`、`2=DEC`、`3=INC`、`4=SWAP`、`6=DUP`、`8=SUB`、`9=ADD`、`a=MUL`、`e=DONE`。仓库 `solver.py` 的助记符表误把 8、9 的加减名称交换，但其正式生成路径从不发出 ADD/SUB，所以官方 exploit 不受该笔误影响；自行扩展求解器时应按 Verilog 修正。

直接用增量构造 32 位素数会超限。对目标 $p$ 同时分解 $p-1$ 和 $p+1$，选择最大素因子更小的一侧：

$$
p=\left(\prod_i q_i^{e_i}\right)\pm1.
$$

2、3、5、7、11、13 用手工优化的短指令序列生成；每个因子再用 `DUP` 复制到所需次数，最后连续 `MUL`。若出现大于 13 的素因子 $q$，就递归对 $q-1$ 或 $q+1$ 做同样分解。最后根据所选的是 $p-1$ 还是 $p+1$，执行一次 `INC` 或 `DEC`：

```python
minus = factorint(prime - 1)
plus = factorint(prime + 1)
factors = minus if max(minus) < max(plus) else plus

# 递归生成各素因子，复制指数次数，再把栈顶因子全部相乘。
program += "MUL\n" * (sum(factors.values()) - 1)
program += "DEC\n" if factors == plus else "INC\n"
program += "DONE\n"
```

服务端生成素数时也调用同一分解器，只接受组装后短于 85 个半字节的实例，因此这条构造保证落在限制内。提交组装后的十六进制串，栈顶等于目标素数时得到：

```text
glacierctf{Stup1d_s3xy_St4ckm4ch1n3s}
```

## 方法总结

这是自定义栈机的程序合成题，决定性工作是恢复指令语义并把大整数改写成短乘法链。对目标的相邻数做因数分解，通常比逐位或逐次递增构造短得多；递归使用 $q\pm1$ 又能继续压缩较大的素因子。由于没有物理接口、采样信号或启动链，本题按主障碍归 Reverse。
