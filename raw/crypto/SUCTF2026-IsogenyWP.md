# SUCTF2026-Isogeny

## 题目简述

题目实现了 CSIDH 风格的交换群作用。有限域模数 `p` 为 511 位，私钥 `pvA/pvB` 是奇素数度同源对应的小指数向量，`cal(A, sk)` 从 Montgomery 曲线

$$
E_A:y^2=x^3+Ax^2+x
$$

出发执行私钥作用，并返回结果曲线 Montgomery 模型的参数。

菜单提供：

- 获取 `pkA=cal(0,pvA)` 与 `pkB=cal(0,pvB)`；
- 输入两条曲线参数，计算 `A=cal(inputA,pvB)`、`B=cal(inputB,pvA)`，输出 `A >> 200`；
- 用真实共享参数 `cal(cal(0,pvB),pvA)` 的十进制字符串做 SHA-256，作为 AES-ECB key 加密 flag。

泄漏接口即使发现 `A != B`，也只是打印 `Illegal public key!`，随后仍输出 `A` 的高位。因此可以固定第二个输入为诚实 `pkB`，向第一个输入注入与 `pkA` 具有特定小度数同源关系的曲线，取得多个相关共享曲线参数的高位。

这正是 [Solving the Hidden Number Problem for CSIDH and CSURF via Automated Coppersmith](https://eprint.iacr.org/2023/1409) 定义的 CI-HNP：给定两条公开曲线以及泄露共享曲线最高若干位的 oracle，恢复完整共享曲线。论文证明，对 CSIDH 泄露超过 $13/24+\varepsilon\approx54.17\%$ 的最高位时，可用自动化 Coppersmith 在多项式时间恢复；本题泄露 511 位参数中的高 311 位，比例约 60.86%，超过该阈值。

## 解题过程

### 1. 构造 `pkA` 的两个 4-isogenous 邻居

对 $E_A$，由横坐标为 $1$ 和 $-1$ 的阶 4 点取商，可得到两个 4-isogenous Montgomery 参数：

$$
B=2\frac{A+6}{2-A}\pmod p,
\qquad
C=2\frac{A-6}{A+2}\pmod p.
$$

总 PDF 把它们称为“2-isogenous 邻居”，这是不准确的；这里使用的是论文 CI-HNP/CSIDH 模型中的两个 4-isogenous 邻居。三个参数两两满足：

$$
AB+2A-2B+12\equiv0\pmod p,
$$

$$
CB+2B-2C+12\equiv0\pmod p,
$$

$$
AC-2A+2C+12\equiv0\pmod p.
$$

题目私钥作用仅由奇素数度同源构成，与上述固定的 4-isogeny 交换。因此，先对公开 `pkA` 构造邻居 `pkA_B/pkA_C`，再分别施加 `pvB`，结果仍保持同样的三条代数关系。

### 2. 取得三个相关共享参数的高位

先读取诚实公钥 `pkA, pkB`，然后调用三次 gift：

```text
gift(pkA,   pkB) -> shared_A >> 200
gift(pkA_B, pkB) -> shared_B >> 200
gift(pkA_C, pkB) -> shared_C >> 200
```

接口内部虽然还会计算 `cal(pkB,pvA)` 并报告非法公钥，但泄漏值始终来自第一个输入经 `pvB` 作用后的结果，所以三次查询都能取得所需高位。

将输出左移 200 位，记作 $A_0,B_0,C_0$。完整参数可写成

$$
A=A_0+x,qquad B=B_0+y,qquad C=C_0+z,
$$

其中

$$
0\le x,y,z<2^{200}.
$$

### 3. 建立三元模小根方程

把未知低位代入 4-isogeny 关系：

$$
f=(A_0+x)(B_0+y)+2(A_0+x)-2(B_0+y)+12\equiv0\pmod p,
$$

$$
g=(C_0+z)(B_0+y)+2(B_0+y)-2(C_0+z)+12\equiv0\pmod p,
$$

$$
h=(A_0+x)(C_0+z)-2(A_0+x)+2(C_0+z)+12\equiv0\pmod p.
$$

三个未知量的界均为 $2^{200}$。官方利用直接调用 `cuso.find_small_roots([f,g,h], bounds, [p,p,p])`，得到 $(x,y,z)$。

总 PDF 采用论文作者的 [automated-coppersmith](https://github.com/juliannowakowski/automated-coppersmith) 实现。该仓库不仅给出通用 Coppersmith 代码，也提供与上述 $f,g,h$ 完全对应的 `experiments_cihnp/csidh.sage`：根据乘积多项式选取 monomial set，构造 optimal shift polynomials，经 `flatter` 或 Sage/fplll 约化后，用 Gröbner basis 和 CRT 提取三个小根。仓库在没有 `flatter` 时会自动退回 Sage 原生 LLL。

针对本题，参赛脚本作了两点工程处理：

- 先用总参数 `m=6`，失败时再尝试 `m=9`；
- 把 Gröbner 提取阶段允许跳过的失败小素数数量从 100 增至 3000，避免启发式提取过早终止。

无论使用 `cuso` 还是作者实现，恢复后都必须验证 $0\le A,B,C<p$，并重新检查三条模 $p$ 关系，不能只相信格算法返回值。

### 4. 派生密钥并解密

真实共享参数为

$$
shared=A_0+x.
$$

按服务端源码精确使用十进制字符串派生 32 字节密钥：

```python
key = sha256(str(shared).encode()).digest()
pt = unpad(AES.new(key, AES.MODE_ECB).decrypt(ciphertext), 16)
```

不要对大整数使用字节编码，也不要误用十六进制字符串，否则 key 不同。最终得到：

```text
SUCTF{Actu41ly_th1s_iS_4_Pr0blem_7hat_w4s_s0lved_1n_2023_https://eprint.iacr.org/2023/1409}
```

官方利用中的比赛 IP、端口属于临时环境，正文只保留交互顺序；论文和作者实现链接则直接支撑攻击模型与工具用法，予以保留。

## 方法总结

- gift 的致命问题不是单纯泄露高位，而是“非法公钥仅告警、不拒绝”，使攻击者能查询具有已知 4-isogeny 关系的相关输入。
- 奇素数 CSIDH 群作用与 4-isogeny 交换，因此输入端的三条关系会被传递到未知共享曲线端。
- 三份高位泄漏把问题转化为三个 200 位有界未知量的模小根方程；本题 60.86% 的泄露比例超过论文给出的 CSIDH $13/24$ 阈值。
- Coppersmith 返回结果后要复核取值范围、三条同源关系和最终 AES 明文，避免把启发式 Gröbner 提取的伪根当成共享秘密。
