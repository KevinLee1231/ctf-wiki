# ezDLP

## 题目简述

题目给出模合数 `n` 上的矩阵离散对数问题。附件脚本生成随机矩阵 `a`，取大素数 `k`，计算 `b = a ** k`，再用 `md5(long_to_bytes(k))` 作为 AES-ECB 密钥加密 flag。公开数据为 `n, a, b` 的 Sage 序列化文件 `data.sobj` 和一段 Base64 密文。

题目定义代码的关键部分如下：

```python
n = 144709507748526661267852152217031923282704243254105275252262414154410511284347828603240755427862752297392095652561239549522158121842455510674435510821274029842500154931546666242034086499872050823824437303603895977092291834159890433746969317535636398062008995784281741721729948231010601796589449187553147904043991226174291329
a = Matrix(Zmod(n), getRandomMatrix())

k = getPrime(1000)
b = a ** k
save([n, a, b], "data.sobj")

key = hashlib.md5(long_to_bytes(k)).digest()
ciphertext = AES.new(key, AES.MODE_ECB).encrypt(flag)
```

题目提示里说 `n` 的分解可以自行获取。由于矩阵行列式满足 `det(b) = det(a)^k`，矩阵 DLP 可以转成标量 DLP。

## 解题过程

先加载 `data.sobj` 得到 `n, a, b`，分解 `n = p * q`。对两边取行列式：

```text
det(b) = det(a ** k) = det(a)^k (mod n)
```

于是分别在模 `p` 和模 `q` 下求：

```text
det(a)^k = det(b)
```

每个素数域上的离散对数可用 Pohlig-Hellman。`p - 1`、`q - 1` 分解后，对每个素因子幂做 BSGS，再 CRT 合并：

```python
def bsgs(G, kG, p, order):
    t = int(sqrt(order)) + 2
    table = {}
    tG = pow(G, t, p)
    cur = 1
    for a in range(t):
        table[cur] = a
        cur = cur * tG % p

    invG = pow(G, -1, p)
    cur = kG
    for b in range(t):
        if cur in table:
            return t * table[cur] + b
        cur = cur * invG % p

def pohlig_hellman(G, kG, p, order):
    mods, vals = [], []
    for q, exp in factor(order):
        pe = q ** exp
        mods.append(pe)
        vals.append(bsgs(pow(G, order // pe, p), pow(kG, order // pe, p), p, pe))
    return crt(vals, mods)

def dlp(G, kG):
    kp = pohlig_hellman(G, kG, p, p - 1)
    kq = pohlig_hellman(G, kG, q, q - 1)
    return crt([kp, kq], [p - 1, q - 1])

k = dlp(a.det(), b.det())
```

恢复 `k` 后，计算 AES key 解密密文：

```python
key = hashlib.md5(long_to_bytes(k)).digest()
plain = AES.new(key, AES.MODE_ECB).decrypt(base64.b64decode(ciphertext))
```

## 方法总结

- 核心技巧：矩阵幂离散对数通过行列式降维为标量离散对数。
- 识别信号：题目给出 `b = a ** k` 且矩阵行列式不是退化值时，先尝试 `det(b)=det(a)^k`。
- 复用要点：模合数上的 DLP 通常先分解模数，在各素数模下求解后 CRT 合并。
