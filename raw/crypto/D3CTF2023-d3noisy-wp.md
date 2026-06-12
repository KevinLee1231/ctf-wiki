# d3noisy

## 题目简述

题目给出若干 3211-bit 大整数在多组 321-bit 素数模数下的打乱余数。目标是恢复这些大整数并组合出私钥解密。核心困难是每一行余数顺序被 shuffle，无法直接 CRT；可把 noisy CRT 转成子集和/背包问题，再用格规约恢复匹配关系。

## 解题过程

最终有 8 个解。

主要内容是中国剩余定理（CRT）和格规约，把 noisy CRT 转化为子集和问题。

```
def leak(N):
    p,S = [],[]
    for i in range(15):
        p.append(getPrime(321))
        r = [N[_]%p[i] for _ in range(15)]
        shuffle(r)
        S.append(r)
    return p, S
```

本题关键是恢复 `N` 中的 3211-bit 大整数，并把它们相加得到私钥后解密。题目给出了若干 321-bit 素数。

同时给出了矩阵的变换，初始形式如下：

$$
\begin{matrix}
N_1\bmod p_1 & N_2\bmod p_1 & \cdots & N_m\bmod p_1 \\
N_1\bmod p_2 & N_2\bmod p_2 & \cdots & N_m\bmod p_2 \\
\cdots & \cdots & \ddots & \cdots \\
N_1\bmod p_m & N_2\bmod p_m & \cdots & N_m\bmod p_m
\end{matrix}
$$

将每一行元素打乱后得到新的矩阵。

### 求解方法

预期解可以参考论文 “Noisy Polynomial Interpolation and Noisy Chinese Remaindering”。论文中对本题有用的思想是：把未知的余数匹配看成 noisy CRT 重构问题。每个候选大整数都对应从每一行模数余数中选取一个值，正确选择会让 CRT 组合结果接近目标大小的整数。这样就能把选择过程编码成子集和风格的格问题。

考虑将其中一个大整数记为，并记录对应选择关系；如果条件成立则取，否则取。

$$
N_k=\sum_{i=1}^{m}\sum_{j=1}^{m}\delta_{ij}S_{i,j}P_iP_i^{-1}\pmod P
$$

也就是说，它可以看成若干项的子集和。根据题目数据，模数位数较小，而目标整数为 3211 bit，因此可以使用背包格做格规约。设定相关参数后，格构造如下：

$$
\begin{bmatrix}
S_{1,m}P_1P_1^{-1}\\
S_{2,1}P_2P_2^{-1}\\
\cdots\\
S_{m,m}P_mP_m^{-1}
\end{bmatrix}
$$

$$
\begin{bmatrix}
B & & & \\
& B & & \\
& & \ddots & \\
0 & 0 & \cdots & B
\end{bmatrix}
$$

只要满足对应界限，目标向量就是短向量。由于每个目标项的格式都与上面的公式相同，格规约后取前 `m` 个向量即可。

非预期解：

由于本题整体规模相对较小，等价规模不高，也可以使用 meet-in-the-middle 这类以空间换时间的优化方法降低复杂度，从而暴力恢复私钥。

利用脚本：

```
from Crypto.Util.number import *
from sage.all import *
from sympy import nextprime
from out import *
nn = n
n = m = 15
B = getPrime(3211)
P = 1
for i in range(n):
    P *= p[i]
L = []
for i in range(n):
    t = inverse(P//p[i],p[i])
    L.append(t*(P//p[i]))
BB = matrix(n*m+1)
BB[0,0] = P
for i in range(n):
    for j in range(m):
        t = i*m + j
        BB[t+1,t+1] = B
        BB[t+1,0] = S[i][j] * L[i]
red = BB.LLL()
pro = 0
for i in range(n):
    pro ^= int(red[i][0])
pro = nextprime(int(pro))
print(pro)
print(long_to_bytes(pow(c,pro,nn)))
#flag = b'antd3ctf{0c85f77e-bfee-da57-78f2-e961ffd4ca45}'
```

## 方法总结

- 核心技巧：打乱余数的 noisy CRT、CRT basis 构造、背包格、LLL 恢复候选大整数。
- 识别信号：多个大整数对多组小素数取模，且每组余数被打乱时，应把“每行选一个余数”建模成子集和，而不是逐行暴力匹配。
- 复用要点：外部论文提供的是 noisy CRT 到格问题的建模思路；本题参数较小，也存在 meet-in-the-middle 等非预期恢复路径。
