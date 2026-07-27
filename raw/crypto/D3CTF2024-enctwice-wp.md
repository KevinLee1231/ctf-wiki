# enctwice

## 题目简述

服务最多允许取得 8 组加密结果。每组结果可拆成 `ct1 || val || iv || nonce`，其中中间整数满足近似公因子结构：

$$
val_i=q_iX+r_i
$$

$X$ 是服务端隐藏的大整数，$r_i$ 相对较小；在 flag 密文中又有

$$
tag=val\bmod X,\qquad ct_2=\left\lfloor\frac{val}{X}\right\rfloor
$$

服务还提供 `change X` 功能，但错误使用了 `eval` 解析输入；提交 `D3` 可以把 $X$ 调整到适合格攻击的规模。恢复 $X$ 后，解题主线转为利用服务返回的 padding 有效性，逐字节恢复第二层明文。

## 解题过程

### 通过 AGCD 恢复隐藏公因子

从每次加密结果中取 `ct[32:-28]`，将其按大端整数解释为 $val_i$。`change X` 处提交 `D3`，再收集 8 个随机 31 字节明文的密文；把 flag 对应的 $val$ 也加入样本。

第一阶段格构造从样本中恢复与小误差对应的商，第二阶段用 LLL 找到 $X$。下面保留决定解法的矩阵结构；`a` 是各个 $val_i$：

```sage
from sage.all import ZZ, matrix

def recover_small_parts(a):
    n = 7
    E = 2**250
    N = 2**1024
    lattice = matrix(ZZ, n + 1)
    recovered = [0] * n

    for i in range(n):
        lattice[i, i] = E
        lattice[i, n] = a[i]
    lattice[n, n] = N

    for i in range(n - 1):
        lattice[i, n] = a[n - 2]
        lattice[n - 2, n] = a[i]
        reduced = lattice.LLL()
        hermite = reduced[:-2].hermite_form()

        scale = 1
        for j in range(n - 2):
            scale *= hermite[j, j] // E

        recovered[n - 1] = abs(hermite[n - 2, n - 2] // E) * scale
        recovered[i] = abs(hermite[n - 2, n - 1] // E) * scale

        lattice[i, n] = a[i]
        lattice[n - 2, n] = a[n - 2]

    return recovered

def find_X(a):
    n = 7
    E = 2**250
    B = 2**300
    small_parts = recover_small_parts(a)
    lattice = matrix(ZZ, n + 2)
    lattice[0, 0] = 1
    lattice[1, 1] = B

    for i in range(n):
        lattice[0, i + 2] = small_parts[i] * (B // E)
        lattice[1, i + 2] = a[i] * (B // E)
        lattice[i + 2, i + 2] = 2**1024

    return abs(lattice.LLL()[0, 0])
```

该问题本质上是 AGCD（Approximate Common Divisor）：多个大整数共享公因子 $X$，但每个样本都带有小余数。`change X` 的表达式注入并不直接泄露 flag，而是把参数调整到格攻击可以稳定恢复的范围。

### 构造 padding oracle

得到 $X$ 后先拆出 flag 密文的两个字段：

```python
tag = val % X
ct2 = long_to_bytes(val // X)
```

服务在解密后会区分 padding 是否有效，因此可以按标准 CBC padding-oracle 思路，从末尾向前恢复字节。假设当前已经猜到目标明文后缀 `flag`，`expect` 表示希望解密后形成的合法 padding，则应提交。下面是交互核心伪代码，其中 `send`、`oracle_returns_valid` 和题目字段拆分函数需要按服务协议实现：

```python
for length in range(31, 0, -1):
    flag = pad(bytes(length))
    for index in range(length, 0, -1):
        expect = pad(flag[:index - 1])
        for guess in range(1, 128):
            flag = flag[:index - 1] + bytes([guess]) + flag[index:]
            forged_ct2 = xor(xor(flag, expect), ct2)
            forged_val = tag + bytes_to_long(forged_ct2) * X
            forged = ct1 + long_to_bytes(forged_val) + iv + nonce
            send(forged)
            if oracle_returns_valid():
                break
```

每次命中 `Valid` 就固定一个字节，最终去除 padding。官方题解验证得到：

```text
d3ctf{eabd797469b8f88a788a0a04}
```

## 方法总结

- 核心技巧：先用 AGCD 格攻击恢复隐藏公因子 $X$，再利用可重组的 `val=tag+ct2*X` 构造 padding oracle。
- 识别信号：多组大整数共享未知大因子且只有小余数、样本数受限、服务又能调整公共参数时，应检查 AGCD/LLL；解密接口区分 padding 错误时应继续考虑 oracle。
- 复用要点：表达式注入有时只用于改善密码分析参数，而不是直接代码执行；恢复 $X$ 后必须保持 `tag` 不变，仅替换商对应的 `ct2`。
