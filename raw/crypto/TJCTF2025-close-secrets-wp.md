# close-secrets

## 题目简述

题目实现了一套自定义 Diffie-Hellman 加密。公开参数为 $p,g,u=g^a\bmod p,v=g^b\bmod p$，共享秘密为 $k=v^a\bmod p$。真正的缺陷不是离散对数一般情形可解，而是私钥 `a` 由 `randint(p - 10, p)` 产生，只可能落在 11 个值中。flag 先倒序并与 `SHA256(str(k)).hexdigest()` 的 ASCII 字节循环异或，再把每个中间字节 $x$ 变为 $(x+k\bmod 256)\cdot k$。

## 解题过程

枚举 $a\in\{p-10,\ldots,p\}$，用公开关系 $g^a\bmod p=u$ 检验即可恢复私钥。随后计算共享秘密，并按相反顺序撤销两层变换：每个密文整数先整除 $k$，再减去 $k\bmod256$；接着与哈希十六进制字符串的字节异或，最后把结果整体反转。

```python
import ast
import hashlib

params = {}
with open("params.txt", "r", encoding="utf-8") as f:
    exec(f.read(), {"__builtins__": None}, params)

p, g, u, v = (params[name] for name in ("p", "g", "u", "v"))
with open("enc_flag", "r", encoding="utf-8") as f:
    ciphertext = ast.literal_eval(f.read())

a = next(
    candidate
    for candidate in range(p - 10, p + 1)
    if pow(g, candidate, p) == u
)
k = pow(v, a, p)

offset = k % 256
stage1 = [(value // k) - offset for value in ciphertext]
key = hashlib.sha256(str(k).encode()).hexdigest().encode()
plaintext = bytes(
    value ^ key[i % len(key)]
    for i, value in enumerate(stage1)
)[::-1]
print(plaintext.decode())
```

脚本在附件参数上恢复出：

```text
tjctf{sm4ll_r4ng3_sh0rt_s3cr3t}
```

## 方法总结

- 核心技巧：利用私钥与公开大素数 $p$ 的极小距离，把离散对数退化为 11 次枚举。
- 识别信号：即使模数很大，只要指数来自短区间，安全性仍由区间大小而不是整数位数决定。
- 复用要点：先恢复共享秘密，再逐层逆运算；乘法层能够整除是很强的正确性检查，错误候选通常无法得到合理字节。
