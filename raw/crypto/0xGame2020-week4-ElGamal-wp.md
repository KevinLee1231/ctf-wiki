# week4ElGamal

## 题目简述

题目把 flag 转成二进制位，并把每个 `0` 编码为模 $p$ 的随机二次剩余、每个 `1` 编码为随机二次非剩余，再分别进行 ElGamal 加密。由于 ElGamal 的乘法结构会泄露明文的勒让德符号，可以逐位恢复 flag。

## 解题过程

ElGamal 公钥为 $(p,g,y)$，其中 $y=g^x\bmod p$。加密随机数为 $k$ 时，密文为

$$
c_0=g^k\bmod p,\qquad c_1=m\,y^k\bmod p.
$$

对非零元素定义勒让德符号 $\chi(a)\in\{1,-1\}$。由欧拉判别准则：

$$
a^{(p-1)/2}\equiv
\begin{cases}
1 & a\text{ 是二次剩余},\\
-1\pmod p & a\text{ 是二次非剩余}.
\end{cases}
$$

因此

$$
\chi(c_1)=\chi(m)\chi(y)^k.
$$

题目给出的实际参数满足 $\chi(y)=1$，所以 $\chi(c_1)=\chi(m)$：`c1` 为二次剩余就对应明文位 `0`，否则对应 `1`。无需私钥，也无需使用 `c0`。

```python
from Crypto.Util.number import long_to_bytes

p = 10946148224653120484646906462803901217745837751637974066354601688874051778651193811412739372059281847771491564589986518154039493312147458591216351424346123
y = 2101136318398982764494355697982735290351867853540128399809061806690701481465143258501856786165972388085070268979718711434744226290744692988395355120277617


def legendre(value: int) -> int:
    result = pow(value % p, (p - 1) // 2, p)
    if result == 1:
        return 1
    if result == p - 1:
        return -1
    return 0


assert legendre(y) == 1

bits = []
with open("data", "r", encoding="ascii") as stream:
    for line in stream:
        if not line.strip():
            continue
        c0_hex, c1_hex = (part.strip() for part in line.split(",", 1))
        c1 = int(c1_hex, 16)
        symbol = legendre(c1)
        if symbol == 0:
            raise ValueError("密文分量不应为 0")
        bits.append("0" if symbol == 1 else "1")

bit_string = "".join(bits)
value = int(bit_string, 2)
flag = long_to_bytes(value)
print(flag.decode())
```

若某组正确生成的参数满足 $\chi(y)=-1$ 且 $\chi(g)=-1$，则 $\chi(c_0)=(-1)^k$，应改用

$$
\chi(m)=\chi(c_0)\chi(c_1).
$$

不过本仓库源码存在两处不一致：给定的 $g$ 与 $y$ 都是二次剩余，`g` 的任意幂仍是二次剩余，所以尝试在 `while` 中生成非剩余局部变量 `y` 不会成功；而且这个局部变量也没有写回 `key.y`，实际加密仍使用原公钥。仓库没有收录运行时生成的 `data` 文件，因此最终字节只能用比赛附件运行上述脚本恢复，不能从 `task.py` 单独重建。

## 方法总结

漏洞来自明文直接落在“二次剩余/非剩余”两个可公开判定的集合中。先用欧拉判别准则确定实际公钥 $y$ 的勒让德符号，再决定只看 $c_1$，还是联合 $c_0,c_1$ 消去随机数奇偶性。分析必须以实际参数为准，不能照搬源码中未生效的公钥修改逻辑。
