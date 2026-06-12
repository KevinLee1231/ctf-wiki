# f, l and ag++

## 题目简述

题目源自 `f, l and ag`，但增加了 10 秒时间限制。解法使用现成的高阶多项式/resultant 与快速插值技巧，在 `Zmod(N)` 上恢复 `f`、`l`、`ag` 相关量并拼回 flag。

## 解题过程

### 关键观察

题目源自 `f, l and ag`，但增加了 10 秒时间限制。

### 求解步骤

出题人：拿了一道现成的题目（https://github.com/soon-haari/my-ctf-challenges/tree/main/2
024-dreamhack-invitational/f%2C l and ag），加上了 10 秒的时间限制
我：拿了一份现成的代码，加上了 IO 交互
Neobeo 的代码只需要 5 秒，符合题目要求
解题代码（修改自 Neobeo 的代码）：
import os
os.environ["TERM"] = "linux"
from pwn import process, remote
# io = process(["python3", "server.py"])
io = remote("<challenge-host>", 42104)
from Crypto.Util.number import getPrime, GCD, bytes_to_long
for i in range(6): exec(io.recvline().decode())

F.<x> = Zmod(N, is_field=True)[]
l2, ag2, flag2 = [F(z)/f_enc for z in [l_enc, ag_enc, flag_enc]]

### 参考链接补充

原题 `f, l and ag` 的参考解法把 `f`、`l`、`ag` 记作 `A`、`B`、`C`，把完整 flag 记作 `D`，并利用三段拼接长度固定这一点建立关系：

- `D = 256^51 * A + 256^34 * B + C`，其中 `A`、`B` 为 17 字节，`C` 为 34 字节。
- 在 `Zmod(N)` 中除以 `C` 后令 `a = A / C`、`b = B / C`，得到 `(256^51 * a + 256^34 * b + 1)^257 = D_enc / C_enc`，同时有 `a^257 = A_enc / C_enc`、`b^257 = B_enc / C_enc`。
- 参考解法的核心不是直接做二元多项式 gcd，而是用变形的 Franklin-Reiter 思路逐步消去 `a`，把问题化成只含 `b` 的多项式；消元过程中每轮把系数模 `f3(b)` 约回低次数，避免常数项膨胀。
- 得到 `b` 后可恢复 `a`。因为 `A`、`C` 都远小于 `N`，再用 LLL 从 `a = A / C mod N` 中恢复真实小整数 `A`、`C`，最后恢复 `B` 并拼回 flag。

本题 `++` 版本只额外增加 10 秒限制，因此复用 Neobeo/原题思路时，真正需要优化的是 resultant/插值和 IO，而不是更换数学模型。

### PDF 图片

![Discord 中给出的 solve.sage 附件线索](RCTF2025-f-l-and-ag-plusplus-wp/discord-solve-script-attachment.png)

![Discord 讨论中提到 resultant 是整题核心的提示](RCTF2025-f-l-and-ag-plusplus-wp/discord-resultant-hint.png)

### PDF 外链

- <https://github.com/soon-haari/my-ctf-challenges/tree/main/2024-dreamhack-invitational/f%2C>
- <https://soon.haari.me/flandag/>

### 跨页补回：resultant 与插值脚本收尾

##############################################################################
## Fast resultant calculation
@parallel(ncpus=20)
def get_resultant(i):
    B = x^e - ag2
    A = (x+256^34*i)^e - flag2 - B
    t = 1
    while A.degree():
        t *= B.lc()
        A,B = B,A%B
    return F(i)^e, t^2 * A[0]^B.degree()

arr = [a for _,a in get_resultant([0..e])]
print('Finished calculating resultants')

##############################################################################
## Fast lagrange interpolation
poly = vector(CRT_basis([x-i for i,_ in arr])) * vector(j for _,j in arr)
expanded_poly = F([j for i in poly for j in [i]+[0]*(e-1)])

##############################################################################
## Calculate everything else to get flag
r = -gcd(expanded_poly, (x-256^17)^e-l2)[0]
s = -gcd(x^e-ag2, (x+256^34*r)^e-flag2)[0]
l,f = rational_reconstruction(r-256^17, N).as_integer_ratio()
ag = ZZ(f*s)
flag = (b''.join(int(z).to_bytes(17,'big') for z in [f,l]))
flag += (b''.join(int(z).to_bytes(34,'big') for z in [ag]))
print(flag.hex())

io.sendline((hex(bytes_to_long(flag))[2:]).encode().hex().encode())
io.interactive()
# RCTF{hah_i_only_wanted_to_steal_your_high_degree_speedrun_techniques__:P}

## 方法总结

- 核心技巧：resultant、CRT basis 和快速 Lagrange interpolation。
- 识别信号：高次数多项式关系但只需恢复少数结构化未知量。
- 复用要点：时间限制题优先复用成熟高速实现，再只补 IO 交互。
