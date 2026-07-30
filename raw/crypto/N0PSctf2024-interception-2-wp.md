# Interception 2

## 题目简述

`blocks.txt` 共有五行。前两行表面上也是十六进制整数，实际将开头的 `0x` 去掉并按字节解码后，分别得到十六进制表示的 RSA 公钥参数 `n` 和 `e`；后三行才是 RSA 密文块。

公钥指数 `e` 与模数 `n` 的位数相近。结合题目要求“截获并解密消息”，应优先检查私钥指数 $d$ 是否过小，从而满足 Wiener 攻击的条件。

## 解题过程

### 解析公钥和密文块

前两行可以这样还原：

```python
lines = open("blocks.txt", encoding="ascii").read().splitlines()

n_text = bytes.fromhex(lines[0][2:]).decode()
e_text = bytes.fromhex(lines[1][2:]).decode()

n = int(n_text.split("=", 1)[1], 16)
e = int(e_text.split("=", 1)[1], 16)
ciphertexts = [int(line, 16) for line in lines[2:]]
```

RSA 中有 $ed-k\varphi(n)=1$。当 $d$ 过小时，$\frac{k}{d}$ 会出现在 $\frac{e}{n}$ 的连分数收敛分数中。枚举这些收敛分数，并检查由候选 $\varphi(n)$ 推导出的二次方程是否有整数根，即可恢复私钥。

以下脚本不依赖第三方 Wiener 攻击库，能够直接处理题目附件：

```python
from math import isqrt


def continued_fraction(numerator: int, denominator: int):
    while denominator:
        yield numerator // denominator
        numerator, denominator = denominator, numerator % denominator


def convergents(terms):
    p_prev2, p_prev1 = 0, 1
    q_prev2, q_prev1 = 1, 0

    for term in terms:
        p = term * p_prev1 + p_prev2
        q = term * q_prev1 + q_prev2
        yield p, q
        p_prev2, p_prev1 = p_prev1, p
        q_prev2, q_prev1 = q_prev1, q


def wiener_attack(e: int, n: int) -> int | None:
    for k, d in convergents(continued_fraction(e, n)):
        if k == 0 or (e * d - 1) % k:
            continue

        phi = (e * d - 1) // k
        p_plus_q = n - phi + 1
        discriminant = p_plus_q * p_plus_q - 4 * n
        if discriminant < 0:
            continue

        root = isqrt(discriminant)
        if root * root == discriminant and (p_plus_q + root) % 2 == 0:
            return d
    return None


lines = open("blocks.txt", encoding="ascii").read().splitlines()
n = int(bytes.fromhex(lines[0][2:]).decode().split("=", 1)[1], 16)
e = int(bytes.fromhex(lines[1][2:]).decode().split("=", 1)[1], 16)

d = wiener_attack(e, n)
assert d is not None
print(f"d = {d}")

plaintext = bytearray()
for line in lines[2:]:
    message = pow(int(line, 16), d, n)
    plaintext.extend(message.to_bytes((message.bit_length() + 7) // 8, "big"))

print(plaintext.decode())
```

恢复出的私钥指数为：

```text
33355850749750804374792378057688740300670580694053668626680293995787819532569
```

三段明文拼接后是一封组织成员之间的会面通知，其中给出了一条当时用于传递下一次会面指令的音频地址，并在末尾直接包含：

```text
N0PS{r5A_w13n3R_4T74cK}
```

音频地址是一次性比赛资源，不影响本题密码攻击，也不是复现所需依赖，因此不保留该外链。

## 方法总结

- 核心技巧：从异常大的 RSA 公钥指数反推“小私钥指数”风险，枚举 $\frac{e}{n}$ 的连分数收敛分数恢复 $d$。
- 识别信号：题目同时给出 `n`、`e` 和密文，常规分解不可行，但 $e$ 与 $n$ 同量级。
- 复用要点：候选收敛分数不能直接视为私钥；必须用判别式检验候选 $\varphi(n)$ 是否对应整数素因子。解密分块时还要按大端整数还原字节，再按原顺序拼接。
