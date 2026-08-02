# N1CTF 2021 - ctfhub

## 题目简述

题目把 Web 业务与一个经过 `-Ofast` 优化的原生 PHP 扩展组合起来。扩展实现了 64 位分组、CBC 工作模式的自定义密码；编译器内联和展开循环后，反编译结果接近千行。Web 端允许通过修改 cookie 调用解密路径，预期解法是把它当作选择密文解密 oracle，对三轮 Feistel 结构做差分分析并恢复 64 位密钥，再伪造 PHP 序列化用户对象读取任意文件。

虽然最后一步发生在 Web 中，决定性障碍是自定义分组密码的密钥恢复，因此归入 Crypto。

## 解题过程

### 从重复代码还原分组密码

仓库的 `crypt.cpp` 给出了清晰版本。每个 64 位分组先经过置换并分成两个 32 位半块，随后执行三轮 Feistel：

$$
L_{i+1}=R_i,\qquad R_{i+1}=L_i\oplus f(R_i,K_i).
$$

轮函数先把 32 位输入扩展为 48 位，经过置换、与 48 位轮密钥异或，再进入 8 个 $6\rightarrow4$ 位 S 盒。64 位主密钥通过固定置换产生三轮密钥。分组外层使用 CBC，首块 IV 为 0。

反编译时无需逐行理解展开后的代码。先识别重复的“扩展—置换—异或—查 S 盒”片段，再恢复三轮循环和 CBC 链，即可得到与 `crypt.cpp` 中 `encblock/decblock` 等价的模型。

### 用解密 oracle 攻击末轮子密钥

Web 的分享功能允许提交可控 cookie 并观察解密结果，相当于 `decrypt_blk(ciphertext)` oracle。对内部置换后的两个密文半块记为 $(L_0,R_0)$，三轮解密可写成：

$$
\begin{aligned}
L_1&=R_0\oplus f(L_0,K_2),&R_1&=L_0,\\
L_2&=R_1\oplus f(L_1,K_1),&R_2&=L_1,\\
L_3&=R_2\oplus f(L_2,K_0),&R_3&=L_2.
\end{aligned}
$$

固定 $L_0$，只改变 $R_0$。oracle 返回明文后重新应用内部置换，可以从两次结果得到进入最后一轮 $f$ 的输入差分，以及扣除已知 $R_0$ 差分后的输出差分。由于 $f$ 由 8 个独立 S 盒组成，每个 S 盒可分别建立差分表：

```python
def differential_table(box):
    table = {}
    for a in range(64):
        for b in range(64):
            table.setdefault((a ^ b, box[a] ^ box[b]), set()).update((a, b))
    return table
```

对某个 S 盒，已知输入值 $u$、输入差分 $\Delta u$ 和输出差分 $\Delta v$。差分表给出所有可能的 S 盒输入，再与 $u$ 异或即可得到该 6 位轮密钥候选。官方 `exp.py` 连续取 7 组随机差分并对候选集合求交，8 个 S 盒通常都只剩一个候选，从而恢复 $K_0$ 的 48 位。

主密钥还有 16 位未进入这一轮密钥。枚举 $2^{16}$ 种尾部，并用两个已知密文/明文对验证三轮解密：

```python
for tail in range(1 << 16):
    candidate = recovered_48_bits + bits_of(tail, 16)
    if all(dec_blk(c, candidate) == p for c, p in checks):
        master_key = candidate
        break
```

### 伪造用户 cookie 读取 flag

取得主密钥后即可离线解密、修改并重新加密 CBC cookie。官方 README 给出的业务利用链是：

1. 构造一个伪造的 PHP `User` 序列化对象。
2. 利用 `User.__toString()` 枚举 `/tmp/box/_user`，定位实际保存 flag 的随机目录。
3. 将 cookie 中的 `userinfo` 路径改为攻击者伪造的用户文件。
4. 再次提交经过正确加密的序列化对象，把路径指向目标目录中的 flag 文件，实现跨目录任意文件读取。

作者还提到 CBC padding oracle 可能构成非预期解，但没有验证；本文不把未确认分支当作正式解法。仓库未保留远端输出和最终 flag，因此只记录已经由源码与官方脚本支持的完整利用机制。

## 方法总结

面对被编译器严重内联的密码实现，先恢复结构而不是逐条追踪指令：分组大小、轮结构、S 盒边界和工作模式一旦确定，差分分析就能按 S 盒拆分。此题还展示了典型跨层利用链：原生扩展泄露密码 oracle，密码密钥恢复后又转化为 PHP 对象与路径控制。
