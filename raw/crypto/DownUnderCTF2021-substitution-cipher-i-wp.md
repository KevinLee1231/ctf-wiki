# DownUnderCTF 2021 - Substitution Cipher I

## 题目简述

加密器逐字符读取明文字节值 $m$，计算公开二次多项式
$f(m)=13m^2+3m+7$，再把结果作为 Unicode 码点输出。字符之间没有状态、密钥或随机性，因此每个密文字符都可以独立反解。

```sage
def encrypt(msg, f):
    return ''.join(chr(f.substitute(c)) for c in msg)

f = 13*x^2 + 3*x + 7
```

## 解题过程

对密文字符 $C$ 取码点 $c=\operatorname{ord}(C)$。对应明文值满足：

$$
13m^2+3m+7-c=0.
$$

例如明文 `a` 的值是 97，对应 $f(97)=122615$。反过来求方程的整数根会得到一个正整数根 97 和一个无效负根。由于 flag 使用普通字节，只保留位于 $0$ 到 $255$ 范围内的整数根即可。

```sage
P.<x> = PolynomialRing(ZZ)
f = 13*x^2 + 3*x + 7

ciphertext = open('../challenge/output.txt', encoding='utf-8').read().strip()
plaintext = []

for char in ciphertext:
    roots = (f - ord(char)).roots()
    candidates = [int(root) for root, multiplicity in roots
                  if 0 <= root < 256]
    if len(candidates) != 1:
        raise ValueError('unexpected root set')
    plaintext.append(chr(candidates[0]))

print(''.join(plaintext))
```

输出为：

```text
DUCTF{sh0uld'v3_us3d_r0t_13}
```

## 方法总结

这是一种把单字节代入公开函数的确定性替换密码。判断重点不是密文显示成罕见 Unicode 字符，而是源码对每个字符独立执行同一个可逆方程。遇到此类变换时，先把字符还原为整数，再解 $f(m)=c$ 并按明文字节范围筛选根；消息长度、字符位置和重复模式都会被完整保留。
