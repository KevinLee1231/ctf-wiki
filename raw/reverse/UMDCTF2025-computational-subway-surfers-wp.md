# UMDCTF 2025 - computational-subway-surfers

## 题目简述

附件只有一个约 3.5 MB 的 `css.html`，没有 JavaScript。页面用 CSS 选择器、
`:has()`、`:hover` 和暂停/继续的 animation 模拟一台 4 位单元的自定义 LC2
机器：HTML 节点表示指令、操作器和磁带，CSS 动画负责移位、读写、进位和跳转。

虚拟机内部又实现了 S-box 状态更新、$p=2^{127}-1$ 上的 Montgomery 曲线标量乘法、
汉明距离校验和一层二进制线性变换。虽然解法需要 SageMath 中的密码学运算，但参赛
附件首先要求从 CSS 恢复虚拟机及其程序语义，因此按决定性载体归入 `reverse`。

## 解题过程

### 1. 从 CSS 还原检查程序

仓库中的生成源码表明，随机类名只是隐藏语义；核心对象仍分为 program、operation
以及 ROM、临时区和 flags 等 tape。把 HTML 中的指令节点按地址复原后，可以得到
`code.asm`。程序先验证 `UMDCTF{...}` 的固定格式，随后处理 232 个 4 位输入值
（索引 `0` 到 `231`）。

主循环维护一个 127 位状态 `t12`。每轮从输入取得两个 nibble `r1`、`r2`，通过固定
256 项置换表更新状态：

```python
def hash1(seed, r1, r2):
    new_seed = 0
    for j in range(32):
        out = prng_table[((seed >> (4*j)) & 0xf) + (r1 << 4)]
        new_seed |= (out & 0xf) << (4*j)
        r1 = (out >> 4) ^ r2
    return new_seed
```

然后计算曲线点横坐标：

```text
x_Q = x(new_seed * Q)
x_P = x(new_seed * P)
```

`x_Q` 的低 16 位被压成两个 4 位值，与嵌入的 `spine1`、`spine2` 比较并累计汉明
距离；`x_P` 则成为下一轮状态。最终只允许总误差小于 2。生成器故意翻转了
`spine1[22]` 的一位，所以解码器必须容忍恰好这一处噪声。

### 2. 恢复被 puncture 的 58 个约束位

曲线为：

```text
y^2 = x^3 + 62406*x^2 + x  (mod 2^127-1)
```

已知横坐标 `P = 3`、`Q =
74991407119409573653134810242842117674`。程序末尾的特殊曲线检查实际约束
`P = qQ`，其中 `q` 只有 58 位。官方 `solve.sage` 先乘曲线 cofactor，把可能的低
2 位余类分开，再在宽度约 $2^{57}/4$ 的区间内用有界离散对数搜索恢复其余部分：

```python
for b in range(4):
    if Q1 * b == P1:
        break

q = 4 * bounded_dlog(P - b*Q, 4*Q, (0, 2^57 // 4)) + b
puncturing = [(q >> i) & 1 for i in range(58)]
```

这 58 位分别约束特定轮次输入 nibble 的最低位，大幅缩小后续搜索空间。

### 3. 用 beam search 还原 232 个 nibble

从初始状态 `0x7331` 开始，每轮枚举 `r1,r2` 的 256 种组合，并应用对应的
puncturing 位约束。候选经 `hash1` 更新后，计算 `x(new_seed * Q)` 与 spine 的
汉明距离；累计代价大于 1 的路径立即剪枝。脚本还向前试探下一轮，删除不可能继续
匹配的状态。

116 轮后得到 232 个 nibble。按低位优先展开为 924 bit，再逆转生成器中的
924 阶 GF(2) 线性变换矩阵：

```python
lc2inv = Matrix(GF(2), rows) ** (-1)
char_bits = lc2inv * vector(GF(2), recovered_bits)
```

每 7 位对应一个字符索引。生成器对每个位置都用固定随机种子打乱了可打印字符表，
所以还需用同一位置的 `charbits` 映射把索引换回实际字符。最终得到：

```text
UMDCTF{F34r/n0t'tH3-Man*Wh0+h@s%wR1773n|t3n*TH0us4nd$J4v45Cr1pt//fUnC7i0n5,Bu7^Th3&M4N|wH0>ha5+Wr1tt3N\0ne~CSS<An1mATi0n:10,000.Tim3s##~^$*}
```

## 方法总结

本题要分层处理，不能直接在 3.5 MB CSS 中搜索 flag：先把 CSS 动画还原成虚拟机
指令，再把汇编归纳为状态转移与曲线检查，随后恢复短离散对数约束，最后用低误差
beam search 和 GF(2) 逆矩阵解码。有界离散对数是官方 Sage 解法的主要计算开销，
复现时应单独预留运行时间，而不是把它和后续轻量解码混在一起反复执行。
