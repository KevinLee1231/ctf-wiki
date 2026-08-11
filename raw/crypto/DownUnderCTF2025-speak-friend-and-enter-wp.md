# Speak Friend, and Enter

## 题目简述

服务给出随机 48 字节 challenge，要求提交 JSON 的 RSA `public_key` 与对该 challenge 的签名。它不固定公钥本身，而是只验证

$$
\operatorname{CMAC}_{AES}(\operatorname{long\_to\_bytes}(public\_key))
=\texttt{9d4dfd27cb483aa0cf623e43ff3d3432},
$$

再要求公钥整数恰为 2048 bit。通过后，服务按裸 RSA 检查 $signature^{65537}\equiv\operatorname{SHA512}(challenge)\pmod {public\_key}$。

题目完整源码 `cryptosecrets.py` 给出了 AES-CMAC 密钥 `2b7e151628aed2a6abf7158809cf4f3c`。因此 MAC 不是公钥承诺，而是攻击者可计算的固定密钥 MAC；可以构造一个已知分解的 2048 bit 合数，使其恰好具有服务端期待的 CMAC。

## 解题过程

### 构造指定标签的 CMAC 输入

CMAC 对完整最后块 $M_r$ 的计算可写为

$$
C_{r-1}=AES_K(\cdots),\qquad
T=AES_K(C_{r-1}\oplus M_r\oplus K_1),
$$

其中 $K_1$ 由 $AES_K(0^{128})$ 左移并按 CMAC 规则条件异或 $\texttt{0x87}$ 得到。给定任意块对齐的前缀 $P$ 和目标标签 $T$，先按 CBC-MAC 计算其链值 $C_{r-1}$，再令

$$
M_r=AES_K^{-1}(T)\oplus C_{r-1}\oplus K_1.
$$

这样 `P || M_r` 的 CMAC 就是服务端标签。官方 solver 以 16 个 `ff` 字节开头、接 224 个随机字节作为 $P$，再补出 16 字节 $M_r$。总长度为 256 字节且首字节非零，所以 `long_to_bytes(public_key)` 保持这 256 字节不变，并满足 2048 bit 检查。下面是公式对应的辅助函数骨架，`cbc_mac_state` 与 `cmac_subkey_1` 需要按前述 CMAC 规则实现。

```python
prefix = b"\xff" * 16 + os.urandom(224)  # 15 个完整块
state = cbc_mac_state(aes_key, prefix)
k1 = cmac_subkey_1(aes_key)
last = strxor(strxor(AES.new(aes_key, AES.MODE_ECB).decrypt(target_tag),
                    state), k1)
forged_n = bytes_to_long(prefix + last)
assert forged_n.bit_length() == 2048
assert CMAC.new(aes_key, ciphermod=AES).update(long_to_bytes(forged_n)).digest() == target_tag
```

### 选择可分解的伪造模数

CMAC 约束并不保证 `forged_n` 的因子已知，所以不断改变前缀直到它满足官方 solver 的可分解筛选：用小素数试除，剩余因子须为素数；所有因子互异且不含 2。实际实现还应检查 $\gcd(65537,\varphi(forged_n))=1$，才能计算私钥指数。

若因子列表为 $f_1,\ldots,f_t$，则

$$
\varphi(forged_n)=\prod_{j=1}^{t}(f_j-1),
\qquad d=65537^{-1}\bmod\varphi(forged_n).
$$

虽然这个模数不一定是传统的两个素数乘积，服务端仅以 `RSA.construct((public_key, 65537))` 构造公开 RSA 对象，随后只做模幂；它没有验证模数来源或因子数。

### 对 challenge 做裸 RSA 签名

读取 challenge 后，将其 SHA-512 摘要解释为整数 $s$。由于 $s$ 只有 512 bit，小于 2048 bit 的伪造模数，直接计算

```python
h = bytes_to_long(SHA512.new(challenge).digest())
signature = pow(h, d, forged_n)
payload = {"public_key": forged_n, "signature": signature}
send_json(payload)
```

便有 $signature^{65537}\equiv h\pmod{forged_n}$，同时前一阶段的 CMAC 比较也成立。

### 验证

官方 `solve.py` 在找到满足因子筛选的 `new_pub` 后，计算 $d$、读取 challenge 并提交 JSON。题目配置给出的验证材料为 `DUCTF{Z3n_th3_fl@g_Out!}`。本归档没有运行随机搜索；CMAC 补块公式、素因子筛选和签名流程均来自题目源码与官方 solver 的静态对照。

## 方法总结

- 核心技巧：已知 CMAC 密钥时，可以为任意块对齐前缀反解最后一个完整块，使标签等于指定值；把这种“可选择 MAC 输入”与可分解 RSA 模数结合即可伪造签名。
- 识别信号：认证逻辑若只比较某个整数的 MAC，却没有固定该整数或验证其结构，且 MAC 密钥来自公开测试向量、源码泄露或其他可恢复来源，应检查 chosen-input MAC forgery。
- 复用要点：CMAC 的最后块要区分完整块使用 $K_1$ 和非完整块使用 $K_2$；还要保证整数序列化长度不变。构造可分解模数时必须同时满足位数、奇偶性、平方因子和 $\gcd(e,\varphi)=1$ 条件。
