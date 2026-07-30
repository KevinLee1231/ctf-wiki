# L3akCTF 2024 Henny on the Rocks Writeup

## 题目简述

服务使用 NIST P-521 曲线实现 ECDSA，并允许获取公钥、对任意消息签名以及取得 AES-CBC 加密的 flag。ECDSA 私钥同时被散列为 AES 密钥：

```python
KEY = randint(1, n - 1)
Q = KEY * G
AES_KEY = hashlib.sha256(long_to_bytes(KEY)).digest()

def get_k():
    return int.from_bytes(
        hashlib.sha512(os.urandom(64)).digest(),
        "big"
    ) % n
```

P-521 子群阶约为 521 位，而 SHA-512 的输出只有 512 位，所以模运算实际上不会扩展随机数；每个签名 nonce 的最高 9 位恒为零。这种部分 nonce 泄漏可以转化为 Hidden Number Problem，并通过格规约恢复 ECDSA 私钥。

## 解题过程

ECDSA 签名满足：

$$
s_i \equiv k_i^{-1}(h_i+r_i d)\pmod n
$$

等价地：

$$
k_i \equiv s_i^{-1}h_i+s_i^{-1}r_i d\pmod n
$$

其中 $d$ 是固定私钥，$k_i<2^{512}$。相对于 521 位的 $n$，每个样本都给出了 $k_i$ 的 9 个最高有效位。收集足够多组签名后，这些近似同余式共同约束同一个 $d$。

官方脚本重复对字符串 `message` 请求签名，记录 $(r_i,s_i)$，并把 nonce 描述为：

```python
known_nonce = PartialInteger.from_bits_be(
    "000000000" + "?" * 512
)
```

随后将消息散列 $h_i$、签名分量和部分已知 nonce 送入 `dsa_known_msb`。该实现构造 HNP 格并执行短向量搜索，通常约 80 至 100 个签名即可得到私钥候选。候选不能仅凭数值范围接受，还应验证：

$$
dG=Q
$$

验证通过后，按照服务端相同方式派生 AES 密钥。加密结果前 16 字节是 IV，其余部分是 CBC 密文：

```python
aes_key = hashlib.sha256(long_to_bytes(int(d))).digest()
blob = bytes.fromhex(encrypted_flag)
iv, ciphertext = blob[:16], blob[16:]
plaintext = unpad(
    AES.new(aes_key, AES.MODE_CBC, iv).decrypt(ciphertext),
    AES.block_size,
)
```

仓库中的服务端 flag 可验证结果为：

```text
L3AK{9_b1ts_12_m0r3_th4n_3n0ugh}
```

## 方法总结

- 核心技巧：把 ECDSA nonce 的固定高位转化为 Hidden Number Problem，通过格规约恢复私钥。
- 识别信号：曲线阶比 nonce 生成器输出更长，尤其是 P-521 配合 512 位哈希或 PRNG 输出时，应立即比较二者位长。
- 复用要点：格攻击得到的候选必须用公开点 $Q=dG$ 验证；恢复签名私钥后，还要检查它是否被复用于对称加密、认证或其它协议环节。
