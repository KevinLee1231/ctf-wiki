# Veiled XOR

## 题目简述

题目直接给出 RSA 实例中的三个公开量：

1. 模数 `n = p \cdot q`；
2. 密文 `c = m^e mod n`（`e = 65537`）；
3. 隐写掩码 `veil = p ^ reverse_bits(q)`。

其中 `p,q` 均为 1024 位素数，`veil` 是将 `q` 按二进制位序逆转后的整数与 `p` 按位异或得到。

`admin/sol.py` 提供了完整复现实战路线，因此本题核心不在「发现接口」，而在于从这两个可编排约束反推出 `p,q`。

## 解题过程

### 关键观察

令 `r = reverse_bits(q)`，已知

$$
veil = p \oplus r
$$

又有

$$
n = p \cdot q
$$

这是一对耦合约束：`r` 的每一位对应 `q` 的镜像位，`p` 与 `q` 同时约束乘积。`admin/sol.py` 用的策略是逐位 DFS 构造 `p`、`q`，并持续用 `n` 的模约束剪枝：

$$
(p \cdot q) \bmod 2^{k} = n \bmod 2^{k}
$$

以及 `0 < p\cdot q \le n` 的上界/下界约束，结合位掩码关系逐步确定两数全部位。

### 关键步骤

`admin/sol.py` 代码里先把 `a = veil` 转为 1024 位二进制字符串 `bina`，然后递归：

1. `dfs(P, Q, Round)` 按比特位逐步向下扩展 `P`,`Q`；
2. `Round` 每前进 1 位，就约束两位：
   - `CurP` 对应 `p` 的一位；
   - `CurQ` 对应 `q` 的对应镜像位（由 `bina[Round]` 与 `bina[1023-Round]` 的异或关系给定）；
3. 每步先做边界剪枝：如果 `CurP * CurQ > n`、或者最小可达积仍超过 `n`、或低 `Round+1` 比特同余不匹配则回溯；
4. 当 `Round == 512` 且 `P * Q == n` 时返回成功，得到 `p,q`。

递归成功后：

$$
\varphi(n) = (p-1)(q-1),\quad d = e^{-1}\bmod \varphi(n)
$$

再做标准 RSA 反推：

$$
m = c^d \bmod n
$$

```python
from Crypto.Util.number import *

n = ...
c = ...
a = ...  # Veil XOR value
b = bin(a)[2:]
b = '0'*(1024-len(b)) + b

def dfs(P,Q,Round):
    if Round==512:
        if P*Q==n:
            global p,q
            p,q=P,Q
            return 1
    for i in range(2):
        for j in range(2):
            CurP = P+i*(2**(1023-Round)) + (int(b[Round])^j)*(2**Round)
            CurQ = Q+j*(2**(1023-Round)) + (int(b[1023-Round])^i)*(2**Round)
            if CurP*CurQ>n:
                continue
            if  (CurP+2**(1023-Round))*((CurQ+2**(1023-Round)))<n:
                continue
            if (CurP*CurQ)%(2**(Round+1))!=n%(2**(Round+1)):
                continue
            dfs(CurP,CurQ,Round+1)
    return 0
```

```python
phi=(p-1)*(q-1)
d=inverse(e,phi)
print(long_to_bytes(pow(c,d,n)))
```

### 验证

- `dfs` 在 `Round=512` 收敛到与 `n` 精确因式一致的 `(p,q)`；
- 用 `d` 解密 `c` 后得到题面 flag `bi0sCTF{X0rcery_R3ve3rsing_1s_4n_4rt_2d3e3d}`，与 README 标注一致；
- 该流程不依赖额外外链，源自 `admin/sol.py` 的直接逻辑。

## 方法总结

- 关键技巧是“位级约束 + 乘积约束”联合搜索，而不是盲猜 RSA 因子。
- `Veil XOR` 的价值在于把 `q` 的位结构反向绑定到 `p`，从而使模乘同余在递归中可高效剪枝。
- 这类题的通用迁移点是：
  - 先把每个可用约束都转成位级等式；
  - 用逐位 DFS/回溯加同余剪枝降低搜索空间；
  - 最后再回到标准 RSA `n,e,c` 解密链条收口验证。
