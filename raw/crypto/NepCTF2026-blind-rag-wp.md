# NepCTF2026 Blind RAG Writeup

## 题目简述

服务在有限域 $\mathbb{Z}_p$ 上实现 64 维向量的“可搜索加密”。文档向量被随机拆成两份并分别乘秘密矩阵；查询接口则返回用户向量经过两个秘密逆矩阵后的结果。由于查询输入完全可控且令牌原样返回，接口实际上暴露了两个线性变换 oracle。

## 解题过程

系统生成可逆矩阵 $M_1,M_2$。对文档行向量 $v$，随机选择 $v_1$ 并令：

$$
v_2=v-v_1\pmod p.
$$

文档密文为：

$$
c_1=v_1M_1^T,\qquad c_2=v_2M_2^T.
$$

查询向量 $q$ 的令牌却直接计算为：

$$
t_1=qM_1^{-1},\qquad t_2=qM_2^{-1}.
$$

提交第 $i$ 个标准基向量 $e_i$ 时，返回值恰好是 $M_1^{-1}$ 和 $M_2^{-1}$ 的第 $i$ 行。依次查询 64 个标准基即可完整恢复两个逆矩阵：

```python
M1_inv = []
M2_inv = []

for i in range(64):
    q = [0] * 64
    q[i] = 1
    t1, t2 = query_token(q)
    M1_inv.append(t1)
    M2_inv.append(t2)
```

在公开数据库中找到 `label == "flag"` 的记录。由加密公式：

$$
v_1=c_1(M_1^{-1})^T,\qquad
v_2=c_2(M_2^{-1})^T,
$$

所以：

$$
v=c_1(M_1^{-1})^T+c_2(M_2^{-1})^T\pmod p.
$$

恢复 flag 向量后，按题目实现把每个坐标编码为固定 32 字节小端序，再计算 SHA-256：

```python
import hashlib

def vector_to_key(v):
    raw = b"".join(
        (x % (1 << 256)).to_bytes(32, "little")
        for x in v
    )
    return hashlib.sha256(raw).digest()
```

加密字段 `c_d` 的布局为：

```text
nonce (12 bytes) || ciphertext || tag (16 bytes)
```

使用 AES-256-GCM 解密并验证：

```python
from Crypto.Cipher import AES

raw = bytes.fromhex(flag_entry["c_d"])
nonce = raw[:12]
ciphertext = raw[12:-16]
tag = raw[-16:]

key = vector_to_key(flag_vector)
cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
flag = cipher.decrypt_and_verify(ciphertext, tag).decode()
print(flag)
```

结果为：

```text
flag{v3ct0r_bl1nd_r4g_cpa_m4tr1x_r3c0v3ry}
```

题目部署时误用了 `flag{...}` 前缀，这里保留服务实际解密结果。

## 方法总结

随机拆分文档向量并不能弥补查询端的确定性线性泄漏。只要攻击者可选择 $q$ 并观察 $qM^{-1}$，标准基查询就会逐行公开整个秘密矩阵。设计可搜索加密时，查询令牌必须带有不可消除的随机化或受到严格访问限制，不能把秘密线性映射直接作为公开 API。
