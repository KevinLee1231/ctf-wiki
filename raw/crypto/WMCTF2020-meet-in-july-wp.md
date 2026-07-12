# Meet in July

## 题目简述

题目逆向后得到一套 Lucas-RSA 风格校验：程序从二进制中取出 `msg="rdyu@20200701"` 和 `keytail`，计算 HMAC-SHA256 作为 256-bit 目标值，再验证输入 `s` 是否满足 `lucas(s, e, N) == hash`。由于 `N` 只有 257 bit，可用 msieve 分解，进而在 Lucas 群上求私钥指数 `d = inverse(e, lcm(p-1,q-1,p+1,q+1))`，反解出满足校验的 `s`。

## 解题过程

1. 用msieve分解得到N的素因子， N是257bits的Modulus可以几分钟内分解。

N= 1000000000000000000000000000000E98C3C3C3C3C3C3C3C3C3C3C3C3C3C6B15

分解得到两个素因子：

P = F0F0F0F0F0F0F0F0F0F0F0F0F0F0F185

Q = 110000000000000000000000000000051

2. 从程序取出参数：

string msg="rdyu@20200701";

char *keytail = "E98C3C3C3C3C3C3C3C3C3C3C3C3C3C6B15";

并计算：

Hash = sha2_hmac(keytail, msg)

得到hash=

234704797D8535D5BDEFCFC753B935B1676A8DC2D7D63759DE1A6144862F8445



Hash转为256 bits的big number



3. 根据程序验证：

取 flag{s} ， 验证  lucas(s, e, N) == hash， 验证通过则flag正确。

实际是已知明文hash，公钥e,n, 求对应的lucas-RSA密文s。

程序取e=0x7,采用迭代方式计算lucas序列：

如下： Lucas e(s,1) mod N

Lucas sequence 采用如下迭代：

Vi+j = Vi * Vj - Vi-j



根据代码的计算流程，看出来是计算了 lucas V7, 所以得到e=7

Lucas e(S, 1) mod N， 如下计算：

V0 = 2

V1 = S

V2 = S^2 - 2

V3 = V2 * V1 - V1

V4 = V3 * V1 - V2

V7 = V4 * V3 - V1



从而取phi= lcm(p-1, q-1, p+1,q+1) , lcm为最小公倍数。

d = inverse(e, phi) = inverse(7, phi) ,得到d



然后解出s：

s = lucas( hash, d, N )

得到： 26EDE3FE048B6BFA04F647259A3F00505FD9C9CCB87298CD631FD91F17CCB620

## 方法总结

- 核心技巧：从程序中提取 HMAC 参数和 Lucas 指数 `e=7`，分解小模数 `N`，用 `lcm(p-1,q-1,p+1,q+1)` 构造 Lucas-RSA 私钥指数反解。
- 识别信号：校验函数不是普通 `pow(s,e,N)`，而是递推 Lucas 序列 `V_i`，同时模数很小可分解，应联想到 Lucas/RSA 变体。
- 复用要点：先写清 `V0=2, V1=S, V_{i+j}=V_iV_j-V_{i-j}` 的程序语义，再套私钥指数；不要把它误当普通 RSA 直接开方。
