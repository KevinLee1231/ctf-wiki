# apacha

## 题目简述

程序使用 32 位整数数组校验输入。反编译代码中出现常量 `0x9e3779b9`、轮数表达式 `6 + 52 / n` 以及依赖相邻数据和四字密钥的混合函数，这是 XXTEA 的典型结构。密钥为 `{1, 2, 3, 4}`，对内置的 35 个密文整数执行 XXTEA 解密即可恢复 flag。

## 解题过程

XXTEA 的解密需要严格模拟 C 语言 `uint32_t` 的溢出语义。下面的脚本把所有左移、加法和减法结果截断到 32 位，并按程序中的解密循环处理密文：

```python
cipher = [
    0xE74EB323, 0xB7A72836, 0x59CA6FE2, 0x967CC5C1,
    0xE7802674, 0x3D2D54E6, 0x8A9D0356, 0x99DCC39C,
    0x7026D8ED, 0x6A33FDAD, 0xF496550A, 0x5C9C6F9E,
    0x1BE5D04C, 0x6723AE17, 0x5270A5C2, 0xAC42130A,
    0x84BE67B2, 0x705CC779, 0x5C513D98, 0xFB36DA2D,
    0x22179645, 0x5CE3529D, 0xD189E1FB, 0xE85BD489,
    0x73C8D11F, 0x54B5C196, 0xB67CB490, 0x2117E4CA,
    0x9DE3F994, 0x2F5AA1AA, 0xA7E801FD, 0xC30D6EAB,
    0x1BADDC9C, 0x3453B04A, 0x92A406F9,
]

key = [1, 2, 3, 4]
delta = 0x9E3779B9
mask = 0xFFFFFFFF
n = len(cipher) - 1
rounds = 6 + 52 // (n + 1)
total = (rounds * delta) & mask
y = cipher[0]

while total != 0:
    e = (total >> 2) & 3

    for p in range(n, 0, -1):
        z = cipher[p - 1]
        mix = (
            (((z >> 5) ^ ((y << 2) & mask))
             + ((y >> 3) ^ ((z << 4) & mask))) & mask
        ) ^ (((total ^ y) + (key[(p & 3) ^ e] ^ z)) & mask)
        cipher[p] = (cipher[p] - mix) & mask
        y = cipher[p]

    z = cipher[n]
    p = 0
    mix = (
        (((z >> 5) ^ ((y << 2) & mask))
         + ((y >> 3) ^ ((z << 4) & mask))) & mask
    ) ^ (((total ^ y) + (key[(p & 3) ^ e] ^ z)) & mask)
    cipher[0] = (cipher[0] - mix) & mask
    y = cipher[0]
    total = (total - delta) & mask

print(bytes(value & 0xFF for value in cipher).decode())
```

程序原本也是把解密后每个 32 位整数的低字节作为字符输出，结果为：

```text
hgame{l00ks_1ike_y0u_f0Und_th3_t34}
```

## 方法总结

识别 XXTEA 时可以同时观察 `0x9e3779b9`、`6 + 52 / n`、四元素密钥和相邻字的混合关系，不能只凭单个常量下结论。用 Python 复现 C 密码代码时，最容易遗漏的是无符号 32 位回绕；若不在移位、加减处及时掩码，所得明文会完全错误。
