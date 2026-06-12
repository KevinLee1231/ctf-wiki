# d3pack

## 题目简述

题目基于 affine hidden subset sum problem。给定隐藏子集和相关向量，需要恢复二进制/符号向量并解出隐藏参数。解法核心是构造正交格与 completion lattice，先用 LLL/BKZ 找到具有 `{0,1}` 或 `{-1,0,1}` 结构的短向量，再用贪心恢复二进制矩阵，最后在模域上线性求解。

## 解题过程

本题基于 affine hidden subset sum problem[[1]]。给定满足条件的实例，目标是恢复隐藏参数。

解题关键是使用正交格的概念。给定一个格，其正交格定义为：

一个格的 completion 是对应的完成格。显然，原格是其 completion 的子格。

### 求解方法

对本题，记一个格由给定向量和附加向量共同生成，另一个格只由这些向量生成。

为了解出本题，第一步是计算 completion lattice。

记相关矩阵分块满足对应关系，并有：

按如下方式构造格：

可以证明该格就是目标格的正交格。

然后计算 LLL 规约基，并取前若干个基向量得到目标子空间。

接着使用 BKZ 算法[[1][2]] 计算后续格，LLL 规约基的前若干个基向量就是需要的结果。

最后使用论文[[1]] 中描述的贪心算法恢复隐藏结构，再在模域上解线性方程得到最终参数。

利用脚本：

```
# CRYPTO 2020 hidden subset sum 论文：
# 论文给出了正交格/completion lattice 构造，以及下面使用的贪心恢复策略。
from Crypto.Util.number import *
def allpmones(v):
    return len([vj for vj in v if vj in [-1, 0, 1]]) == len(v)
def orthoLattice(B, x0):
    r, m = B.nrows(), B.ncols()
    M = identity_matrix(m).change_ring(ZZ)
    B1 = B[:, :r]
    B2 = B[:, r:]
    V = Matrix(Zmod(x0), B1)
    W = V.inverse()
    M[r:m,:r] = -(W * B2.change_ring(Zmod(x0))).change_ring(ZZ).transpose()
    M[:r, :r] = x0 * identity_matrix(r)
    return M
def allones(v):
    if len([vj for vj in v if vj in [0, 1]]) == len(v):
        return v
    if len([vj for vj in v if vj in [0, -1]]) == len(v):
        return -v
    return None
def recoverBinary(M5):
    lv = [allones(vi) for vi in M5 if allones(vi)]
    n = M5.nrows()
    for v in lv:
        for i in range(n):
            nv = allones(M5[i] - v)
            if nv and nv not in lv:
                lv.append(nv)
            nv = allones(M5[i] + v)
            if nv and nv not in lv:
                lv.append(nv)
    return Matrix(lv)
def kernelLLL(M):
    n = M.nrows()
    m = M.ncols()
    if m < 2 * n:
        return M.right_kernel().matrix()
    K = 2 ^ (m//2) * M.height()
    MB = Matrix(ZZ, m + n, m)
    MB[:n] = K * M
    MB[n:] = identity_matrix(m)
    MB2 = MB.T.LLL().T
    assert MB2[:n, : m - n] == 0
    Ke = MB2[n:, : m - n].T
    return Ke
n = 50
m = 180
p= # from output.txt
h= # from output.txt
e= # from output.txt
e = vector(e)
h = vector(h)
print("n =", n, "m =", m)
E = matrix(ZZ, 2, m, [e.list(), h.list()])
M = orthoLattice(E, p)
t = cputime()
M2 = M.LLL()
print("LLL step1: %.1f" % cputime(t))
MOrtho = M2[: m-n-1]
print(" log(Height,2)=",int(log(MOrtho.height(),2)))
t2 = cputime()
ke = kernelLLL(MOrtho)
print(" Kernel: %.1f" % cputime(t2))
print(" Total step1: %.1f" % cputime(t))
beta = 2
tbk = cputime()
while beta < n:
    if beta == 2:
        M5 = ke.LLL()
    else:
        M5 = M5.BKZ(block_size=beta)
    # we break when we only get vectors with {-1,0,1} components
    if len([True for v in M5 if allpmones(v)]) == n:
        break
    if beta == 2:
        beta = 10
    else:
        beta += 10
print("BKZ beta=%d: %.1f" % (beta, cputime(tbk)))
t2 = cputime()
MB = recoverBinary(M5)
print("  Recovery: %.1f" % cputime(t2))
L = matrix(Zmod(p), MB).transpose()[:n+1]
L = L.augment(-e[:n+1].column())
h_ = h[0:n+1]
a = L^-1*h_
a = [long_to_bytes(ZZ(i)).strip() for i in a]
print(a[-1])
```

### 参考资料

[1] Coron-Gini 的 CRYPTO 2020 论文给出 affine hidden subset sum 的多项式时间攻击框架：通过正交格/完成格把隐藏的 0/1 结构转成短向量问题，格规约后用贪心过程恢复隐藏子集。本题的 `orthoLattice`、`kernelLLL`、`recoverBinary` 就是围绕这一路线实现。

[2] Nguyen-Stern 对 Qu-Vanstone/Merkle-Hellman 类背包密码的群分解攻击说明了类似“从背包结构中恢复隐藏分解”的思想，作为本题正交格攻击的历史背景。

[1] Coron J. S. 和 Gini A.，《求解 hidden subset sum problem 的多项式时间算法》，CRYPTO 2020，论文地址：https://eprint.iacr.org/2020/461.pdf。

[2] Phong Q. Nguyen 和 Jacques Stern，《重新审视 Merkle-Hellman：基于群分解的 Qu-Vanstone 密码系统分析》，CRYPTO 1997。

论文地址：https://link.springer.com/content/pdf/10.1007/BFb0052236.pdf。

## 方法总结

- 核心技巧：affine hidden subset sum、orthogonal lattice、completion lattice、LLL/BKZ 提取二进制结构、模线性方程恢复参数。
- 识别信号：题目给出大量线性组合向量和隐藏 0/1 子集结构时，应考虑正交格而不是直接枚举子集。
- 复用要点：外链论文的必要信息是“怎样把隐藏子集和转成正交格短向量问题”，不是单纯调用 BKZ；实现时还要检查规约结果是否只含 `{0,1}` 或 `{-1,0,1}` 分量。
