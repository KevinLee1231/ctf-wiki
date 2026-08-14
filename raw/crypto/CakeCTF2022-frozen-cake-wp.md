# CakeCTF 2022 frozen cake Writeup

## 题目简述

题目把整个 flag 转成整数 $m$，生成两个 512 位素数 $p,q$ 和模数 $n=pq$，随后给出：

$$
a=m^p\bmod n,\qquad b=m^q\bmod n,\qquad c=m^n=m^{pq}\bmod n.
$$

目标是在不知道 $p,q$ 的情况下恢复 $m$。这不是模数分解题；三个指数之间已经泄露了足够的代数关系。

## 解题过程

先计算 $ab$ 的逆元，再与 $c$ 相乘：

$$
\begin{aligned}
c(ab)^{-1}
&=m^{pq-p-q}\pmod n\\
&=m^{(p-1)(q-1)-1}\pmod n\\
&=m^{\varphi(n)-1}\pmod n.
\end{aligned}
$$

flag 整数小于 512 位素数，必然与 $n$ 互素。由欧拉定理 $m^{\varphi(n)}\equiv1\pmod n$，因此：

$$
c(ab)^{-1}\equiv m^{-1}\pmod n.
$$

再求一次模逆即可取回明文：

$$
m\equiv\left(c(ab)^{-1}\right)^{-1}\pmod n.
$$

对应脚本很短：

```python
with open("output.txt", "r", encoding="utf-8") as f:
    n = int(f.readline().split(" = ")[1])
    a = int(f.readline().split(" = ")[1])
    b = int(f.readline().split(" = ")[1])
    c = int(f.readline().split(" = ")[1])

m_inv = c * pow(a * b, -1, n) % n
m = pow(m_inv, -1, n)

length = (m.bit_length() + 7) // 8
print(m.to_bytes(length, "big").decode())
```

运行结果为：

```text
CakeCTF{oh_you_got_a_tepid_cake_sorry}
```

## 方法总结

看到同一个明文在同一模数下使用多个相关指数时，应优先检查指数的整数线性组合。这里 $pq-p-q=\varphi(n)-1$，恰好把未知素数隐藏在欧拉函数中，并把给出的三个幂组合成了 $m^{-1}$。

这类题不必恢复私钥或分解模数；先化简指数关系，往往比调用通用 RSA 攻击工具更直接。
