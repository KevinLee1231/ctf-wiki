# Matrix3

## 题目简述

题目实现了论文 ePrint 2023/1745 的矩阵方案，但把推荐参数中的 $n$ 改为 $140$，超过 $10^2=100$。源码中生成 $k=140$ 个小矩阵 $A_i$，再用可逆矩阵 $E$ 共轭：

```python
p = 2**302 + 307
k = 140
n = 10
alpha = 3
D = E * A_i * E**(-1)
```

服务端只要求提交一个矩阵 $E'$，使得所有 $E'^{-1}D_iE'$ 的元素都在 $[0,\alpha)$：

```python
for i in range(k):
    if check_M(E_1 * Dlist[i] * E) == 0:
        self._send(b"your private key is wrong")
        return
self._send(flag)
```

因此恢复等价私钥即可。

参考 URL：https://eprint.iacr.org/2023/1745.pdf

## 解题过程

### 关键机制

因为 $k=140>n^2=100$，公钥矩阵数量多于矩阵空间维数，存在非平凡线性关系。对任意系数 $c_i$：

$$
\sum_i c_iD_i=E\left(\sum_i c_iA_i\right)E^{-1}
$$

$A_i$ 元素很小，若找到短向量 $c$ 使 $\sum_i c_iA_i=0$，则可在 $D_i$ 展平矩阵的右核里找到短关系。进一步对这些关系求核，可恢复被打乱顺序的 $A_i$ 展平结果。

恢复顺序时，先解：

$$
\sum_i c_iD_i=I
$$

对应：

$$
\sum_i c_iA_i=I
$$

结果中为 1 的位置就是原矩阵对角线位置。对任意单位位置 $(x,y)$，有：

$$
E(\mathbf{e}_{x,y})E^{-1}
$$

其行/列比值可用于判断该展平位置属于哪一行哪一列，从而恢复顺序。

### 求解步骤

1. 将每个 $D_i$ 展平为长度 $100$ 的向量，组成 $100\times140$ 矩阵。
2. 构造格求右核短向量，得到 $D_i$ 的线性关系。
3. 再对关系求核并 BKZ，恢复元素为 $0,1,2$ 的展平 $A_i$。技巧是先把 $A_i$ 减一，转成 $-1,0,1$，更适合短向量恢复。
4. 利用 $\sum c_iA_i=I$ 定位对角线。
5. 用共轭后矩阵的第一列比值恢复行标记，用第一行/列方程恢复完整 $E'$。
6. 向服务端提交 $E'$ 的 10 行数据。

交互脚本形式：

```python
def Matrix2strlist(M):
    rows = []
    for i in range(10):
        rows.append(" ".join(hex(M[i, j])[2:] for j in range(10)).encode())
    return rows

payload = Matrix2strlist(E1)
for row in payload:
    io.recvuntil(b">")
    io.sendline(row)
```

## 方法总结

- 漏洞来自参数选择：$k>n^2$ 让公钥矩阵之间产生足够多线性关系。
- 不需要恢复原始私钥，只需恢复任意等价共轭矩阵 $E'$。
- 服务端验证范围只检查小矩阵元素，因此等价密钥可直接过检。
