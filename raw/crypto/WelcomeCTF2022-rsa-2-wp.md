# rsa2

## 题目简述

题目以 $e=3$ 的 RSA 加密一封格式固定的邮件。Flag 是邮件中唯一未知的连续字节块，长度在 60 到 80 字节之间；它前后的正文均已知。未知部分相对 2048 位模数较小，可以把完整明文写成未知量的一次多项式，再使用 Coppersmith 小根攻击。

## 解题过程

设已知前缀、后缀分别为整数 $P,S$，未知 Flag 长度为 $l$ 字节，后缀长度为 $s$，未知整数为 $x$，则完整明文为：

$$
M(x)=P\cdot256^{l+s}+x\cdot256^s+S.
$$

密文满足 $M(x)^3-c\equiv0\pmod n$。枚举可能的 $l$，在 $\mathbb Z_n[x]$ 上构造多项式并求小根：

```sage
R.<x> = PolynomialRing(Zmod(n))

for l in range(60, 81):
    M = bytes_to_long(prefix) * 256^(len(suffix) + l)
    M += x * 256^len(suffix)
    M += bytes_to_long(suffix)

    f = M^3 - c
    f /= f.leading_coefficient()  # 化为首一多项式
    roots = f.small_roots(X=256^l, epsilon=0.05)
    if roots:
        print(long_to_bytes(int(roots[0])))
        break
```

官方脚本以同样思路恢复出：

```text
greyhats{If_you_Know_part_of_it_you_know_all_of_it_RZZgrFNxKvHtsFyU6wQj}
```

## 方法总结

低指数 RSA 不会因为“明文很长”就天然安全。只要未知部分足够小且其在明文中的位置已知，就能建立模 $n$ 多项式并寻找小根。复用此方法时要准确保留换行、空格和字节长度；任一已知片段偏移错误都会使真实明文不再是所构造多项式的根。
