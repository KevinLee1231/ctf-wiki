# UMDCTF 2025 - bron coin

## 题目简述

题目实现了一套自定义隐私币：`BronCoin` 用 Poseidon 哈希提交，转账通过 Groth16 证明输入币属于 Merkle 树、序列号未使用、输出承诺正确且金额守恒。持有一枚地址为 Bron、价值至少 500 的已入树币，即可通过 `/bron/query` 取得 flag。

虽然题面包装成“币”，决定性弱点是零知识电路中的 Merkle 路径约束不完备，因此归入密码学而不是按界面或交易名称机械归入区块链。

## 解题过程

Merkle 路径的每层元素保存 `(sibling, direction)`。正常情况下 `direction` 只能是 0 或 1，用来选择：

$$
\operatorname{parent}=
(1-d)H(x,s)+dH(s,x).
$$

但电路把 `d` 分配为普通域元素 `FpVar`，从未约束 $d(d-1)=0$。于是它不是二选一开关，而是两个哈希值之间的任意域线性组合。

先从公开接口取当前 Merkle 树、Bron 公钥和 Groth16 proving key，再任意构造一枚价值 5000、属于自己密钥的假币，记其承诺为 $c$。沿用真实第 0 个叶子的路径和其余层，只需伪造第一层方向值。设真实第一层父节点为 $y$、兄弟节点为 $s$，选择：

$$
d=\frac{y-H(c,s)}{H(s,c)-H(c,s)}.
$$

代回电路可得：

$$
(1-d)H(c,s)+dH(s,c)=y.
$$

因此第一层之后的值与真实路径完全一致，最终仍能得到公开根，尽管 $c$ 从未在树中。

官方解题脚本的关键构造等价于：

```rust
let h0 = poseidon([commitment, sibling]);
let h1 = poseidon([sibling, commitment]);
let d = (real_parent - h0) * (h1 - h0).inverse().unwrap();

path.val = commitment;
path.list[0].1 = d;
```

随后用这枚假币生成合法 Groth16 证明，并把 5000 拆成：

- 一枚地址为 Bron、价值 5000 的输出币；
- 一枚地址为自己、价值 0 的输出币。

电路只检查输入地址与所给私钥一致、序列号关系正确以及输出价值之和等于输入价值；伪造的 Merkle 成员关系使这枚无中生有的输入币通过验证。服务将两个输出承诺加入树后，把序列化的 Bron 输出币提交给 `/bron/query`，即可得到：

```text
UMDCTF{Lebron-honey-my-pookie-bear-I-have-loved-you-ever-since-I-first-laid-eyes-on-you-xoxo-act-like-an-angel-dress-like-crazy-allthegirlsbegirling}
```

## 方法总结

- 核心技巧：利用 Groth16 电路中未布尔化的 Merkle 方向变量，把分支选择变成域上的线性插值，伪造成员证明。
- 识别信号：电路用普通域变量表达布尔选择，却没有 `is_boolean` 或 $d(d-1)=0$ 约束。
- 复用要点：链下代码里的 `if left/right` 语义不会自动进入 R1CS；审计时必须逐项检查范围、布尔性、公开输入绑定和状态约束是否真的被证明。
