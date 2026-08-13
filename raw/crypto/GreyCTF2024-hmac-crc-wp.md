# hmac-crc

## 题目简述

服务端把 `CRC32(key || plaintext || key)` 当作消息认证码，再将 `plaintext || crc` 填充后用固定 key、固定 nonce 的 AES-CTR 加密。挑战要求为十个随机 16 字节明文生成合法密文。固定 CTR 密钥流与 CRC32 都是仿射映射，因此可以用选择明文查询完整模拟加密接口。

## 解题过程

对固定长度 16 字节明文 $x$，CRC32 带有一个常量偏移，而固定 nonce 的 CTR 加密只是与固定密钥流异或；固定的 PKCS#7 填充也只贡献常量。故整个 32 字节输出可写成

$$
E(x)=L(x)\oplus b,
$$

其中 $L$ 是 $\operatorname{GF}(2)$ 上的线性映射，$b=E(0)$。

先查询 128 个单位比特向量和全零向量，总计 129 次，低于服务端的 200 次限制：

```python
basis_ct = []
for bit in range(128):
    value = bytearray(16)
    value[bit // 8] = 1 << (bit % 8)
    basis_ct.append(encrypt(bytes(value)))

zero_ct = encrypt(b"\x00" * 16)
```

若明文中置位集合为 $S$，异或所有 $E(e_i)$ 会得到 $L(x)$，同时附带 $|S|\bmod2$ 份常量 $b$。为了最终始终保留一份 $b$，当置位数为偶数时再异或 `zero_ct`：

```python
def forge(pt):
    result = b"\x00" * 32
    weight = 0
    for bit in range(128):
        if (pt[bit // 8] >> (bit % 8)) & 1:
            result = xor(result, basis_ct[bit])
            weight += 1
    if weight % 2 == 0:
        result = xor(result, zero_ct)
    return result
```

对挑战给出的十个随机明文逐一返回伪造结果，CRC 校验和解密均通过，得到：

```text
grey{everything_is_linear_algebra_a0945v832q}
```

## 方法总结

CRC 只用于检错，不具备抗伪造能力；把秘密前后缀加入 CRC 也不会消除其仿射结构。固定 nonce 的 CTR 又让相同位置始终使用同一密钥流，两层线性或仿射操作叠加后仍可被基向量查询完全学习。消息认证应使用标准 MAC 或 AEAD，并保证 nonce 按规范唯一。
