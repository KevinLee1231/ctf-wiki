# Java Windows Terminal

## 题目简述

网站使用 RS256 JWT 判断用户身份，并提供公钥与密钥生成脚本。脚本不是独立随机选取两个素数，而是先生成 $p$，再令 $q$ 为紧邻 $p$ 的下一个素数。两个因子过于接近，使 RSA 模数可以被 Fermat 分解；恢复私钥后即可签发管理员令牌。

## 解题过程

从 public.key 解析模数 $N$ 和公钥指数 $e$。Fermat 分解利用：

$$
N=a^2-b^2=(a-b)(a+b)
$$

从 $a=\lceil\sqrt N\rceil$ 开始递增，直到 $a^2-N$ 为完全平方数：

~~~python
from math import isqrt

a = isqrt(N)
if a * a < N:
    a += 1

while True:
    b2 = a * a - N
    b = isqrt(b2)
    if b * b == b2:
        p, q = a - b, a + b
        break
    a += 1
~~~

计算 $\varphi(N)=(p-1)(q-1)$ 与 $d=e^{-1}\bmod\varphi(N)$，将参数编码成 RSA 私钥。然后用 RS256 签发内容为管理员身份的 JWT，例如：

~~~json
{"user":"admin"}
~~~

把新 token 写入网站使用的 cookie 或 Authorization 位置，访问受保护页面即可得到：

~~~text
maple{f3rm4t_!n_7h3_m0d3rn_w0r1d}
~~~

## 方法总结

RSA 的安全性不仅要求因子足够大，还要求生成方式不暴露结构。若 $p$ 与 $q$ 很接近，Fermat 分解只需少量迭代。取得因子后应完整重建私钥并使用网站原本声明的 RS256 算法签名，不能依赖 alg=none 或算法混淆等与本题无关的猜测。
