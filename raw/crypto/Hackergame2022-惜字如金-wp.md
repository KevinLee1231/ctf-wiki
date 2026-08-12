# 惜字如金

## 题目简述

题目定义了“惜字如金化”规则，并给出两个经过该规则删字后的 Python 签名脚本：

1. 单词末尾的 `e/E` 若前一字符是辅音，则删除；
2. 连续重复的同一辅音只保留第一个，比较时忽略大小写。

需要恢复 HS384 的 HMAC 密钥，以及 RS384 中生成 RSA 素数的隐藏短语。两问的核心都是从可逆表示约束中恢复密码参数，因此归入密码方向。

## 解题过程

### HS384：枚举被压缩的密钥

惜字如金后的脚本保留了以下约束：

```python
secret = b'ustc.edu.cn'
check_equals(len(secret), 39)
```

当前字符串只有 11 字节，说明有 28 个字符被删除。题目补充说明被删字符均为小写。根据规则，原串只能在已有辅音后补同一个辅音，并可在符合条件的单词末尾补 `e`，所以结构为：

```text
us(s*)t(t*)c(c*)(e?).ed(d*)u.c(c*)n(n*)(e?)
```

枚举各重复次数，使新增字符总数为 28；对每个候选计算 SHA-384，再对十六进制摘要执行同样的惜字如金化，与脚本中残留摘要比较。唯一匹配的原始密钥是：

```text
usssttttttce.edddddu.ccccccnnnnnnnnnnnn
```

把该值填回还原后的脚本，签名就是标准 HMAC-SHA384：

```python
from base64 import urlsafe_b64encode
from hashlib import sha384
from hmac import digest

secret = b"usssttttttce.edddddu.ccccccnnnnnnnnnnnn"
signature = urlsafe_b64encode(digest(secret, data, sha384)).decode()
```

对服务端给出的三个文件分别签名并提交，即可完成第一问。

### RS384：把重复的 32 进制结构写成小根

还原后的脚本用 26 个短语表示数值。字母 `a` 到 `z` 各对应一个由重复字母组成的短语；短语长度决定其在 `phrase_dict` 中映射到的 $0$ 至 $31$。生成 $p$ 时，从第二项开始循环读取 77 个短语，相当于 32 进制数：

```text
BCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZ
```

元音短语末尾的 `e` 会被删除，因此能直接确定 $A=E=I=O=U=31$。三段字母表高度重复，可把未知部分压缩成一个较小的 $x$：

```text
x = 0000000000000000000000000000000000000000000000000000BCDEFGHIJKLMNOPQRSTUVWXYZ
```

于是

$$
p=ax+b,
$$

其中

$$
a=2^{260}+2^{130}+1,
\qquad
b=31(2^{255}+2^{125}).
$$

已知模数 $n=pq$，而 $x$ 的规模约为 $p$ 的三分之一，可以用 Coppersmith 单变量小根恢复 $x$。SageMath 核心代码如下：

```sage
n = Integer("255877945206268685758225801673342"
            "992785361646269587137135214853754"
            "886550982035142794210497165877879"
            "580039847242541662956641303821238"
            "094690165291113510002309824919965"
            "575769641924765055087675446404464"
            "357056205595528275052777855000807")

a = 2^260 + 2^130 + 1
b = 31 * (2^255 + 2^125)
R.<x> = PolynomialRing(Zmod(n))
f = x + b * inverse_mod(a, n)
x0 = f.small_roots(beta=0.5)[0]

p = Integer(a * x0 + b)
q = n // p
assert p * q == n
```

得到 $p,q$ 后，按标准 RSA 计算

$$
d=e^{-1}\bmod (p-1)(q-1),\qquad e=65537,
$$

重建私钥，并使用 PKCS#1 v1.5 与 SHA-384 对三份数据签名即可。

## 方法总结

惜字如金化虽然不是一一映射，但长度、摘要、模数和重复结构提供了足够的附加约束。HS384 将原像空间压缩到可枚举范围，再用摘要筛选；RS384 则把周期性 32 进制结构表示为 $p=ax+b$，转化成 RSA 已知高位/结构化素数的小根问题。处理有损变换时，重点不是盲目补全源代码，而是列出所有可能原像，并寻找能唯一验证候选的密码学约束。
