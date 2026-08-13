# GreyCTF2022 - Permutation

## 题目简述

题目把密钥交换搬到对称群 $S_{5000}$：公开置换 $g$、$A=g^a$ 和 $B=g^b$，再从 $B^a$ 派生密钥。对一般群离散对数看似困难，但单个置换生成的循环子群可按不相交循环分解。

## 解题过程

先把 $g$ 分解成长度为 $L_j$ 的不相交循环。对每个循环任选一个位置，观察同一元素在 $A$ 中被移动了多少步，即得到

$$a\equiv r_j\pmod{L_j}.$$

```sage
residues = []
moduli = []
for cyc in g.cycle_tuples():
    start = cyc[0]
    shift = cyc.index(A(start))
    residues.append(shift)
    moduli.append(len(cyc))

a = CRT_list(residues, moduli)
```

循环长度可能不两两互素，实际实现需使用兼容的广义 CRT，或逐步合并同余并检查公共因子上的一致性。恢复 $a$ 后计算 $B^a$，按生成脚本规定的置换表示派生密钥并解密：

```text
grey{DLP_Is_Not_Hard_In_Symmetric_group_nzDwH49jGbdJz5NU}
```

## 方法总结

群的规模大不代表离散对数一定难。判断安全性要看所处子群的结构；置换的循环分解直接泄露指数对各循环长度的余数。CRT 合并后只需恢复到生成元阶的模意义下，因为 $B^a$ 对同余类代表元不敏感。
