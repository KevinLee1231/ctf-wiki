# r3ticket

## 题目简述

服务生成 128 个独立的 16 位随机整数：

```python
nums = [secrets.randbits(16) for _ in range(128)]
```

玩家先有一次机会查询 `get_num(nums, index)`。随后进行 16 轮挑战：每轮随机生成一个 24 位整数 $x$，计算

$h=\sum_{i=0}^{127}\text{nums}_i^x$

并只给出 $h$ 的前 64 位十进制数字。玩家必须在三秒内回答正确的 $x$。

预期解法分成两部分：

1. 对插值接口发送一个极大的负数，使一次返回值编码全部 128 个秘密数；
2. 已知秘密数后，把幂和的 64 位前缀转成对 $x\log_{10}m$ 小数部分的高精度约束，再用 LLL/CVP 找到 $x$。

## 解题过程

### 1. 插值实现并不是标准的整体有理数计算

查询函数为：

```python
def get_num(nums, index):
    if index > len(nums) - 1:
        return 0
    result = 0
    for i in range(len(nums)):
        part = 1
        for j in range(len(nums)):
            if j == i:
                continue
            part *= (index - j) // (i - j)
        result += part * nums[i]
    return result
```

标准 Lagrange 插值应先把所有分子、分母相乘，再做精确除法。这里却对每一个因子单独使用 Python 向下取整除法 `//`，因此查询点离开 $0,\ldots,127$ 后，返回值会成为另一组整数多项式权重的线性组合。

上界检查只拒绝大于 127 的索引，负数不会被拒绝。令：

$L=\operatorname{lcm}(1,2,\ldots,127)$

$B=L\cdot2^{1000}$

并查询：

$\text{index}=-B$

$B$ 能被所有可能的距离 $|i-j|$ 整除，而且远大于后续系数。

### 2. 把每个错误插值因子写成关于 $B$ 的一次式

固定 $i$ 和 $j$。若 $j<i$，令 $d=i-j>0$，则：

$\left\lfloor\frac{-B-j}{d}\right\rfloor=-\frac{B}{d}-\left\lceil\frac{j}{d}\right\rceil$

若 $j>i$，令 $d=j-i>0$，则：

$\left\lfloor\frac{-B-j}{-d}\right\rfloor=\frac{B}{d}+\left\lfloor\frac{j}{d}\right\rfloor$

因此对每个 $i$，权重

$P_i(B)=\prod_{j\ne i}\left\lfloor\frac{-B-j}{i-j}\right\rfloor$

都是关于 $B$ 的 127 次多项式。查询结果为：

$y=\sum_{i=0}^{127}P_i(B)\,\text{nums}_i$

多项式系数中含有分母。取：

$D=127!$

即可清除所有分母，于是存在整数 $c_0,\ldots,c_{127}$ 满足：

$D\,y=\sum_{k=0}^{127}c_kB^k$

每个 $c_k$ 又是 128 个秘密数的已知线性组合。

### 3. 用平衡进制展开取出 128 个系数

因为 $B$ 含有额外的 $2^{1000}$，远大于所有 $c_k$ 的绝对值。对 $D\,y$ 做以 $B$ 为底的平衡进制展开时，每一位都可以唯一选在区间 $(-B/2,B/2]$：

```python
def balanced_digits(z, base, n):
    digits = []
    for _ in range(n):
        r = z % base
        if r > base // 2:
            r -= base
        digits.append(int(r))
        z = (z - r) // base
    if z:
        raise ValueError("base expansion did not terminate cleanly")
    return digits
```

得到的 128 个 `digits` 就是 $c_0,\ldots,c_{127}$。这一步类似把一次超大整数输出当成一个无进位的 128 槽编码。

### 4. 建立并求解线性系统

对每个 $i$，按上一节的一次因子显式构造 $P_i(q)$，再乘 $D$。把 $q^k$ 的系数放到矩阵第 $k$ 行、第 $i$ 列，就得到已知矩阵 $M$：

$$
\begin{bmatrix}c_0\\c_1\\\vdots\\c_{127}\end{bmatrix}
=M
\begin{bmatrix}\text{nums}_0\\\text{nums}_1\\\vdots\\\text{nums}_{127}\end{bmatrix}
$$

整数矩阵直接求解代价较高，官方脚本改在有限域中求解：

```python
RECOVERY_PRIMES = (65537, 1000003, 1000000007, 1000000009)

for p in RECOVERY_PRIMES:
    F = GF(p)
    rhs = vector(F, [c % p for c in coeffs])
    nums = build_matrix_mod(p).solve_right(rhs)
    nums = [int(x) for x in nums]
    if all(0 <= x < 2**16 for x in nums):
        return nums
```

选择大于 $2^{16}-1$ 的素数后，只要矩阵在该域可逆，模 $p$ 解就直接等于原 16 位整数，不需要 CRT。准备多个素数是为了避开某个素数恰好整除矩阵行列式的情况。

服务在进入 16 轮前会等待 10 秒，官方脚本正是在这段时间完成矩阵构造和秘密数恢复。

### 5. 从十进制前缀得到对数区间

已知全部 `nums` 后，对给定候选 $x$ 可以直接验证前缀，但穷举 $2^{24}$ 个候选无法稳定满足三秒限制。

令：

$m=\max_i\text{nums}_i$

则：

$h=m^x\sum_i\left(\frac{\text{nums}_i}{m}\right)^x$

所以：

$$
\log_{10}h
=x\log_{10}m
+\log_{10}\left(\sum_i\left(\frac{\text{nums}_i}{m}\right)^x\right)
$$

当 $x$ 较大时，第二项迅速趋近于 $\log_{10}t$，其中 $t$ 是最大值 $m$ 出现的次数。

设服务器给出的 64 位前缀对应整数 $P$。若 $\{\log_{10}h\}$ 表示小数部分，则：

$$
\{\log_{10}h\}\in
\left[
\log_{10}P-63,\,
\log_{10}(P+1)-63
\right)
$$

区间非常窄，因此可以约束：

$x\log_{10}m+\log_{10}t\pmod1$

必须落在该区间附近。

### 6. 用格规约寻找 $x$

官方脚本以高精度十进制尺度 $Q=10^p$ 整数化：

$\alpha=\lfloor Q\log_{10}m\rfloor$

把问题写成近似同余：

$x\alpha\equiv Q(\text{目标小数}-\log_{10}t)\pmod Q$

并为二维格使用基：

```python
Matrix(ZZ, [
    [Q, 0],
    [alpha, scale],
]).LLL()
```

第一坐标衡量模 $Q$ 后与目标小数的距离，第二坐标编码 $x$。脚本在规约后的基附近做小范围 CVP 枚举，并另外建立三维格补充候选。对区间内多个采样点重复该过程，再把每个恢复值前后 64 个整数加入候选集。

格方法只负责缩小搜索范围，最终仍使用完整幂和对数验证：

```python
def leading_prefix_logsum(nums, x, digits):
    m = max(nums)
    total = sum((RealField(...)(a) / m) ** x for a in nums if a)
    log_h = x * log10(m) + log10(total)
    return leading_from_log(log_h, digits)
```

保留修正项

$\log_{10}\sum_i(\text{nums}_i/m)^x$

很重要：只用最大项近似虽然能生成候选，却可能在前缀区间边界处误判。

对于很小的 $x$，最大项近似不够好，脚本先用整数精确迭代检查前 5000 个值；若格候选意外失败，还会用递推更新 $(a_i/m)^x$ 的方式扫描一段小范围作为兜底。

### 7. 交互流程

官方 solver 需要 SageMath 和 pwntools。远程运行形式为：

```bash
sage solve/solve.py challenge-host 1337
```

交互步骤为：

1. 收到查询提示后发送 $-L\cdot2^{1000}$；
2. 读取唯一一次查询结果并恢复 128 个 `nums`；
3. 每轮读取 64 位前缀；
4. 由 LLL/CVP 生成少量 $x$ 候选，并以高精度完整对数和逐一验证；
5. 在三秒内返回 $x$，连续完成 16 轮后读取 flag。

## 方法总结

本题的核心是把两个看似信息不足的接口都转化成高精度编码：

- 错误的逐因子 `//` 让一次负数查询变成关于超大基数 $B$ 的整数多项式；平衡进制展开后，一次响应便拆成 128 个线性方程；
- 幂和的 64 位首部等价于 $\log_{10}h$ 小数部分的极窄区间；最大项归一化给出近似线性同余，LLL/CVP 可快速恢复 24 位指数。

解题时既不能把有 `get_num` 名字的函数直接当成正确插值，也不能把“大数只给前 64 位”理解为只泄漏 64 位信息。实现中的取整位置和十进制首位携带的对数小数部分，才是两阶段攻击的真正入口。
