# GreyCTF 2023 AES Confusion

## 题目简述

服务用 Python `random` 生成 AES-CBC 的 flag 密钥和 IV，并为每次加密请求生成新 IV，却没有返回 IV。另一个“解密”功能错误地使用同一密钥做 AES-ECB 解密。两个接口组合后可以泄露每次 CBC 加密使用的 IV，再由连续的 MT19937 输出恢复并回退随机数状态，重建早先生成的 flag 密钥和 IV。

## 解题过程

向加密接口提交空字节串。PKCS#7 会把它填充为 16 个 `0x10`，首个 CBC 密文块为

$C=E_K(P\oplus IV)$。

把同一个 $C$ 交给错误的 ECB 解密接口，得到

$D_K(C)=P\oplus IV$。

因此把响应的 16 字节全部再异或 `0x10` 就是本次随机 IV。`random.randbytes(16)` 由四个连续 32 位 MT19937 输出组成，注意按 Python 的小端字序拆分：

```python
iv = bytes(x ^ 0x10 for x in ecb_plain)
for off in range(0, 16, 4):
    randcrack.submit(int.from_bytes(iv[off:off + 4], "little"))
```

重复 156 次即可提交 $156\times4=624$ 个输出，反 temper 后得到完整 MT 状态。当前状态已经位于所有观测之后，而真正的 key 和 IV 在服务启动时各消耗了四个 32 位字，早于观测序列。官方求解器因此将状态回退：

```python
randcrack.offset(-624 - 8)
key = b"".join(p32(randcrack.predict_getrandbits(32)) for _ in range(4))
iv  = b"".join(p32(randcrack.predict_getrandbits(32)) for _ in range(4))
```

用恢复的 key、IV 对菜单返回的 flag 密文做 CBC 解密并移除填充，得到：

```text
grey{tr4v3ll1n9_84ck_1n_t1m3}
```

## 方法总结

漏洞来自三个接口决策的叠加：用非密码学 PRNG 生成密钥、CBC 不返回 IV，以及允许同密钥 ECB 解密任意密文。空明文让未知量只剩 IV，进而把 PRNG 输出完整暴露。密码材料应来自 `secrets` 或操作系统 CSPRNG，并且不同用途的密钥与解密接口必须严格隔离。
