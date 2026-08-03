# Key in a Haystack

## 题目简述

服务生成一个 40 位素数 `key`，再生成 300 个 1024 位素数，并输出它们与 `key` 的乘积 `haystack`。Flag 经 PKCS#7 填充后，用 `MD5(str(key))` 作为 128 位密钥进行 AES-ECB 加密。虽然乘积有九万余位，但目标因子只有约 12 位，因子规模而不是合数总长度决定了攻击难度。

## 解题过程

先保存服务输出的 `enc_flag` 与巨大整数 $N$。普通试除不现实，但最小素因子仅 40 位，ECM 或 Pollard rho 都适用。官方实例用 `gmp-ecm` 在若干条曲线后找到：

```text
951970865119
```

例如可把只包含十进制 $N$ 的文件交给 ECM：

```bash
ecm -one -c 1000 -inp N.txt 5000
```

这里的 $B_1=5000$ 不是数学保证；若一轮没有命中，应换曲线继续运行。Pollard rho 也是可行备选，其期望复杂度约为 $O(\sqrt p)$，可批量累乘轨道差值并每隔若干轮计算一次 `gcd(product, N)`，减少对超大整数频繁求 gcd 的开销。

得到小因子后，必须按源码中的字节编码还原密钥：对因子的十进制 ASCII 表示取 MD5，而不是对大端整数或十六进制文本取哈希。最后执行 AES-ECB 解密和去填充：

```python
from hashlib import md5
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

small_factor = 951970865119
enc_flag = bytes.fromhex(
    "ce4c5e2f2537a0a215ed30fd9fe6861d"
    "162d01d505d89cdd9255c38cf18ce894"
    "23cc7681c2f7874d698531ff5531893b"
)

aes_key = md5(str(small_factor).encode()).digest()
plaintext = unpad(AES.new(aes_key, AES.MODE_ECB).decrypt(enc_flag), 16)
print(plaintext.decode())
```

输出为：

```text
uiuctf{Finding_Key_Via_Small_Subgroups}
```

## 方法总结

- 不要被模数的总位数误导；整数分解的短板是最小因子的位数。本题特意把一个 40 位素数藏在 300 个 1024 位素数之间。
- ECM 通常更适合从巨大合数中寻找小因子；Pollard rho 也能处理这一规模，并可通过分批 gcd 提升效率。
- 解密阶段应逐字复现源码的密钥派生与填充：`MD5(十进制因子的 ASCII)`、AES-ECB、PKCS#7 去填充，任何表示层差异都会得到错误明文。
