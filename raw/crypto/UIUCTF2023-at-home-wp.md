# UIUCTF 2023 At Home Writeup

## 题目简述

题目给出 `chal.py` 和一组输出 `e`、`n`、`c`。生成过程先随机选取四个 256 位整数 $a,b,a',b'$，再计算：

$$
M=ab-1,\quad e=a'M+a,\quad d=b'M+b,
$$

$$
n=\frac{ed-1}{M},\quad c\equiv flag\cdot e\pmod n.
$$

变量名和大整数外观类似 RSA，但这里并没有使用 $flag^e\bmod n$；密文只是明文整数与 $e$ 的模乘积。因此，决定性问题是判断 $e$ 在模 $n$ 下是否可逆。

## 解题过程

将 $M=ab-1$ 代入 $n$，可得：

$$
n=a'b'ab-a'b'+ab'+a'b+1.
$$

另一方面，

$$
e=a'ab-a'+a=(a'b+1)a-a'.
$$

利用欧几里得算法逐步约去共同项：

$$
\gcd(e,n)=\gcd(a'b+1,e)=\gcd(a'b+1,-a')=\gcd(-a',1)=1.
$$

所以 $e$ 与 $n$ 互素，模逆 $e^{-1}\bmod n$ 必然存在。由题目的密文关系直接得到：

$$
flag\equiv c\cdot e^{-1}\pmod n.
$$

下面的脚本读取题目输出、计算模逆并把结果转回字节串：

```python
from Crypto.Util.number import long_to_bytes

with open("chal.txt", "r", encoding="utf-8") as f:
    values = {
        key.strip(): int(value.strip())
        for key, value in (line.split("=", 1) for line in f)
    }

e = values["e"]
n = values["n"]
c = values["c"]

flag_int = c * pow(e, -1, n) % n
print(long_to_bytes(flag_int).decode())
```

运行后得到：

```text
uiuctf{W3_hav3_R5A_@_h0m3}
```

## 方法总结

这道题借用了 RSA 常见的变量名，但真正的加密关系只是一次可逆模乘。分析密码题时应以实际等式为准，不能根据变量名预设攻击路线。确认 $\gcd(e,n)=1$ 后，使用扩展欧几里得算法或 Python 的 `pow(e, -1, n)` 求模逆即可恢复明文；题目给出的参数构造正是为了保证该模逆存在。
