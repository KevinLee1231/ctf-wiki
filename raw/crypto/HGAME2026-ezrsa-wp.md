# ezRSA

## 题目简述

题目是交互式 RSA oracle。服务端每次连接随机生成 512-bit 的 `p, q`，随机生成 50-bit 的公钥指数 `e`，菜单提供三种功能：加密任意明文并允许翻转 `e` 的某一位、解密任意密文、获取 flag 密文。调用 `Get flag` 后，服务端把 `safe` 置为 `False`，后续加解密输出会经过一层随机异或伪装，但伪装函数不会改变低字节泄露可被利用的本质。

题目关键代码如下：

```python
p = getPrime(512)
q = getPrime(512)
phi = (p - 1) * (q - 1)
e = random.getrandbits(50)
while GCD(e, phi) != 1:
    e = random.getrandbits(50)
d = pow(e, -1, phi)
n = p * q

# Encrypt message
cipher = pow(plain, e ^ (1 << x), n)

# Decrypt message
plain = pow(cipher, d, n)

# Get flag
secret = pow(bytes_to_long(flag), e, n)
safe = False
```

这给了两个核心能力：选择明文下的指数 bit-flip 加密 oracle，以及 flag 密文后的 RSA 解密 oracle。

## 解题过程

第一步恢复 `n`。选择同一个指数翻转位 `x = 0`，分别计算 `2` 与 `4`、`3` 与 `9` 的加密结果。若指数相同，则有：

```text
c(2)^2 == c(4) (mod n)
c(3)^2 == c(9) (mod n)
```

因此取差值 gcd 即可得到 `n` 的倍数，再去除小因子：

```python
c1 = encrypt(2, 0)
c1_ = encrypt(4, 0)
c2 = encrypt(3, 0)
c2_ = encrypt(9, 0)
n = GCD(c1**2 - c1_, c2**2 - c2_)

for i in range(2, 1000):
    while n % i == 0:
        n //= i
```

第二步恢复 50-bit 的 `e`。因为 `phi` 为偶数且 `gcd(e, phi) = 1`，所以 `e` 是奇数，最低位为 1。对明文 `2`，翻转第 `i` 位得到 `2^(e xor 2^i)`。把它乘回 `2^(2^i)` 后与正常 `2^e` 比较，即可判断第 `i` 位是否为 1。

```python
c = c1 * 2 % n
bits = [0] * 50
bits[0] = 1

for i in range(1, 50):
    c_flip = encrypt(2, i)
    if c_flip * pow(2, 1 << i, n) % n == c:
        bits[i] = 1

e = int("".join(map(str, bits[::-1])), 2)
```

第三步做 RSA LSB/低字节 oracle。拿到 flag 密文后，反复把密文乘上 `256^e`，使明文整体左移 1 字节；解密返回值的最低字节给出关于原明文的同余信息。逐轮恢复 `k` 后计算明文：

```python
c = getFlag()
k = 0

for _ in range(127):
    c = c * pow(256, e, n) % n
    p = decrypt(c)
    k = k * 256 + (-inverse(n, 256) * p % 256)

print(long_to_bytes(k * n // 256**127).decode(errors="ignore"))
```

原题解引用的选择明文恢复 `n` 方法可参考 [RSA 选择明文攻击](https://lazzzaro.github.io/2020/05/06/crypto-RSA/#%E9%80%89%E6%8B%A9%E6%98%8E%E6%96%87%E6%94%BB%E5%87%BB)。

## 方法总结

- 核心技巧：选择明文 gcd 恢复 `n`，指数 bit-flip oracle 恢复 `e`，再用解密 oracle 做低字节恢复。
- 识别信号：RSA 服务允许对指数位做可控翻转时，不要只看密钥生成；指数 oracle 本身就可能泄露 `e`。
- 复用要点：恢复 `n` 时常用 `m1^k - m2` 这类同余差值取 gcd；恢复明文时要根据 oracle 输出粒度选择 LSB 或低字节推进。
