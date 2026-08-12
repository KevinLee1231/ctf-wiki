# DownUnderCTF 2021 - Secuchat

## 题目简述

题目给出一份聊天应用的 SQLite 数据库。每个用户记录包含 2048 位 RSA 公钥；每个会话使用 AES-256-CBC，加密会话密钥分别通过双方的 RSA-OAEP 封装。消息发送后还会生成下一组 AES 密钥和 IV，存入关联的 `Parameters` 记录。

生成器为 29 个用户正常生成 RSA 密钥，却额外构造了一个与某位正常用户共享素因子 $p$ 的模数：

```python
common_p = random.choice(random_keys)[1].p
q = getStrongPrime(1024)
vulnerable_n = common_p * q
```

因此问题不是 AES-CBC 或 OAEP 本身，而是 RSA 密钥生成熵失效导致两个公开模数可被最大公约数直接分解。

## 解题过程

从 `User` 表导入所有 RSA 公钥，对模数两两计算：

$$
g_{ij}=\gcd(n_i,n_j).
$$

若 $1<g_{ij}<n_i$，则两个模数共享素因子。数据库只有 30 个用户，直接两两比较已经足够：

```python
import itertools
import sqlite3
from math import gcd
from Crypto.PublicKey import RSA

cur = sqlite3.connect("secuchat.db").cursor()
users = [(name, RSA.import_key(blob))
         for name, blob in cur.execute("SELECT username, rsa_key FROM User")]

for (name_a, key_a), (name_b, key_b) in itertools.combinations(users, 2):
    p = gcd(key_a.n, key_b.n)
    if 1 < p < key_a.n:
        break
```

对任意受影响模数 $n$，令 $q=n/p$，即可重建私钥：

```python
e = 65537
q = key_a.n // p
d = pow(e, -1, (p - 1) * (q - 1))
private_a = RSA.construct((key_a.n, e, d, p, q))
```

然后筛选该用户参与的会话。`initial_parameters` 给出第一把 AES 密钥的两份 OAEP 密文和第一组 IV；根据受害用户是 initiator 还是 peer，选择对应列并解封装：

```python
from Crypto.Cipher import AES, PKCS1_OAEP
from Crypto.Util.Padding import unpad

oaep = PKCS1_OAEP.new(private_a)
aes_key = oaep.decrypt(encrypted_key_for_user)
aes = AES.new(aes_key, AES.MODE_CBC, iv=initial_iv)
```

按时间顺序解密消息。每条 `Message.next_parameters` 指向下一条消息要使用的新 AES 密钥和 IV，因此操作顺序必须是“用当前状态解密本条消息，再用当前记录携带的参数更新状态”：

```python
for ciphertext, wrapped_next_key, next_iv in rows_in_timestamp_order:
    plaintext = unpad(aes.decrypt(ciphertext), AES.block_size)
    print(plaintext.decode())

    next_key = oaep.decrypt(wrapped_next_key)
    aes = AES.new(next_key, AES.MODE_CBC, iv=next_iv)
```

遍历两名共享素因子的用户及其会话后，可以读到：

```text
DUCTF{pr1m1t1v35, p4dd1ng, m0d35- wait, 3n7r0py?!}
```

## 方法总结

批量 RSA 公钥出现共享素因子时，公钥集合本身就泄露私钥；无需攻击 OAEP 或暴力分解 2048 位模数。处理大量 RSA 证书、设备密钥或数据库转储时，应先做批量 GCD 检查。恢复端还必须正确理解应用的密钥状态机：本题每条消息都携带下一轮密钥材料，解密顺序错一条就会使后续 CBC 状态全部失效。
