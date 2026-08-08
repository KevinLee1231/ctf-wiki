# ezECDH

## 题目简述

题目使用曲线 $E:y^2=x^3+x\pmod p$ 做 ECDH，只公开 $G,A,B$ 的 $x$ 坐标，其中 $A=aG$、$B=bG$。共享点为 $S=aB$，其 $x$ 坐标经固定长度编码和 SHA-256 派生 AES-CBC 密钥。参数满足 $p\equiv3\pmod4$，使这条 $j=1728$ 曲线为超奇异曲线；这比一般曲线的离散对数更容易转移到有限域乘法群。

## 解题过程

### 确定可用于配对的子群

在该参数条件下有

$$
\#E(\mathbb F_p)=p+1.
$$

计算 $G$ 的阶 $n$，则 $n\mid p+1$，从而 $p\equiv-1\pmod n$、$p^2\equiv1\pmod n$。因此对该子群的嵌入度至多为 $2$：将曲线扩到 $\mathbb F_{p^2}$ 后，$n$ 阶单位根已存在。实际求解前应确认 $n$ 的可分解性与配对结果的阶；若 $n$ 含有不适合离散对数的巨大因子，不能只因“超奇异”就假定计算可行。

题目只给 $x$ 坐标。对 $G_x,A_x,B_x$ 分别求曲线右端的平方根，可得到 $P$ 与 $-P$ 两个 lift 候选；应保留这些符号分支或在后续 pairing 验证中筛选。

### MOV 转移并求 $a$

在 $E(\mathbb F_{p^2})$ 中选取不与 $G$ 配对退化的 $n$-torsion 点 $Q$，计算 Weil pairing：

$$
\alpha=e_n(G,Q),\qquad \beta=e_n(A,Q).
$$

双线性给出

$$
\beta=e_n(aG,Q)=\alpha^a.
$$

于是把 ECDLP 转为 $\mathbb F_{p^2}^*$ 中的离散对数 $a=\log_\alpha\beta$。用 Sage 等代数系统求离散对数后，必须检查 $aG=A$（允许 $A$ 的符号分支）。若选择的 $Q$ 使 $e_n(G,Q)=1$ 或阶小于 $n$，应换一个 $Q$，不能继续使用该退化配对。

### 计算共享点并解密

对所有通过检查的 $a$ 与 $B$ 符号分支计算 $S=aB$。即使从 $x$ 坐标 lift 时把 $A$ 或 $B$ 取反，得到的标量或共享点可能变号，但

$$
x(S)=x(-S),
$$

故题目只使用共享点 $x$ 坐标时不会受此影响。严格按字段长度编码：

```python
field_bytes = (p.bit_length() + 7) // 8
key = SHA256(xS.to_bytes(field_bytes, "big")).digest()
```

用该 key、题目给出的 IV 和密文执行 AES-CBC 解密，再检查并移除 PKCS#7 padding；正确分支应同时满足 padding 和明文格式。

## 方法总结

- 核心技巧：超奇异曲线的低嵌入度让 MOV 配对把椭圆曲线离散对数移到扩域乘法群。
- 识别信号：$y^2=x^3+x$、$p\equiv3\pmod4$ 以及 $\#E(\mathbb F_p)=p+1$ 是应检查超奇异/MOV 路线的组合信号。
- 复用要点：先验证 $G$ 的子群阶和 pairing 的阶；只给 $x$ 坐标时保留正负分支，并在密钥派生时使用题目规定的固定字节宽度。
