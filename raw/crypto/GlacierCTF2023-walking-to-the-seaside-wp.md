# GlacierCTF2023 - WalkingToTheSeaSide

## 题目简述

服务让 Alice 与 Bob 使用 CSIDH 派生共享曲线，再通过 PBKDF2-SHA512 生成 AES-GCM 密钥。攻击者可以为 Alice 提议新的小素数参数。安全检查按列表长度估计搜索空间，却在真正构造 $p$ 和群作用前用 `set` 去重，没有拒绝重复素数。

## 解题过程

反复提交素数 3，使列表满足

$$
(2\cdot5+1)^{\lvert L\rvert}
$$

达到服务要求的表面安全级别，再追加 5 和 7。检查看到的是很长的列表，实际计算只保留集合 $\{3,5,7\}$，于是

$$
p=4\cdot3\cdot5\cdot7-1=419.
$$

该玩具参数下只有 27 个需要考虑的超奇异 Montgomery 曲线系数。与 Alice 完成交互、提交任意可验证的 Bob 公钥并取得 `{Nonce, CT, Tag}` 后，枚举这 27 个共享曲线值。对每个候选执行与服务相同的 PBKDF2，再用 AES-GCM 验证 tag；错误候选抛出 `ValueError`，唯一通过者给出明文。

```python
for curve in supersingular_curves:
    key = PBKDF2(
        long_to_bytes(curve),
        b"secret_salt",
        count=1_000_000,
        hmac_hash_module=SHA512,
    )
    cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
    try:
        print(cipher.decrypt_and_verify(ct, tag))
        break
    except ValueError:
        pass
```

最终 flag 为：

```text
gctf{always_check_for_duplicate_primes}
```

官方材料还指出一条非预期路径：Alice 没有认证对端确实是 Bob，攻击者可直接冒充 Bob 完成密钥交换。预期解法的重点仍是“验证列表”与“使用集合”之间的参数语义不一致。CSIDH 的群作用背景可参考[原始方案论文](https://csidh.isogeny.org/csidh-20181118.pdf)，但本题所需的降参、枚举和 GCM 验证步骤已在上文完整给出。

## 方法总结

密码参数校验必须针对规范化后的实际参数，而不是未经处理的输入表示。凡是后续存在排序、去重或约简，都应在约简后重新检查安全级别；协议层还应独立验证参与者身份，避免数学安全被认证缺失直接绕过。
