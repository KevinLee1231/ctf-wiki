# NepCTF2026 Leakyrag Writeup

## 题目简述

服务把文档字节映射为 64 维向量，并通过搜索接口返回精确余弦相似度。目标 flag 文档虽然不能直接读取，但其嵌入规则完全公开且确定；只要让它稳定留在 Top 20 结果中，就能用相似度 oracle 逐坐标恢复归一化向量，再反解原始字节。

## 解题过程

对第 $i$ 个字节 $b_i$，未归一化嵌入为：

$$
u_i=\exp\left(\frac{b_i-128}{64}\right),
$$

并使用最后一维 $u_{63}=1$ 作为基准，随后整体归一化。搜索分数为：

$$
s(q)=\frac{\langle v,q\rangle}{\lVert v\rVert\lVert q\rVert}.
$$

附件公开的模板：

```text
flag{l34ky_v3ct0r_s34rch_1s_n0t_3ncrypt10n}
```

虽不是远程真实 flag，但嵌入后可作为 anchor，使带 `[PROTECTED by SecureAI]` 标记的目标文档稳定排在第 9，留在 Top 20 中。记基础查询为 $b$、目标返回分数为 $s_0$。对每一维发送微扰查询：

$$
q_i=b+\varepsilon e_i,
$$

并记录目标分数 $s_i$。由内积的线性关系可得：

$$
\frac{v_i}{\lVert v\rVert}=
\frac{
s_i\lVert b+\varepsilon e_i\rVert
-s_0\lVert b\rVert
}{\varepsilon}.
$$

因此只需一次基线查询和 64 次坐标查询，共 65 次。示意代码：

```python
import math

eps = 1e-3
s0 = score_of_flag_doc(base)
base_norm = l2(base)
recovered = []

for i in range(64):
    q = base.copy()
    q[i] += eps
    si = score_of_flag_doc(q)
    recovered.append(
        (si * l2(q) - s0 * base_norm) / eps
    )
```

若某个微扰让目标掉出 Top 20，就把 $\varepsilon$ 缩小 10 倍后重试该维。整体比例未知不影响解码，因为最后一维对应常量 1。利用坐标比值：

$$
\frac{v_i}{v_{63}}=
\exp\left(\frac{b_i-128}{64}\right),
$$

于是：

$$
b_i=
\operatorname{round}\left(
64\ln\frac{v_i}{v_{63}}+128
\right).
$$

逐字节解码并截取到 flag 结束括号，得到：

```text
NepCTF{11b9d32e-1419-59df-3ea6-02d85f5bafca}
```

## 方法总结

本题虽然以 RAG 为背景，核心却是精确内积 oracle 泄漏。只要服务返回足够精确的相似度，查询向量的微小坐标扰动就等价于数值求梯度，可以恢复目标方向。已知模板的作用只是稳定目标文档的 Top 20 排名；真正还原字节依赖公开的指数嵌入和最后一维基准。
