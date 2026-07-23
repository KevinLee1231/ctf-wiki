# haes

## 题目简述

题目把 AES 扩展成一个 64 字节分组、64 字节密钥的三维版本。状态被排列为 $4\times4\times4$，每轮依次执行 `SubBytes`、自定义 `ShiftPlanes`、按四字节列执行的 `MixColumns` 和轮密钥异或；总轮数只有 5，末轮与 AES 一样省略 `MixColumns`。

服务先返回同一随机密钥下的加密 flag，再提供选择明文加密。一次最多提交 511 个分组。64 字节状态很大，但轮数过少，因此可以使用积分攻击恢复末轮密钥，再逆转密钥扩展得到主密钥。

## 解题过程

### 构造两个积分集合

取一个字节遍历 $0,\ldots,255$，其余 63 字节全为零，得到一个 256 元 $\Lambda$ 集合。为了减少错误末轮密钥字节偶然通过平衡测试的概率，再构造第二个集合：改为让另一个字节遍历全部取值。

两个集合共享全零明文，因此实际只需提交 $256+255=511$ 个分组，恰好不触发服务的长度限制：

```python
lambda_1 = [bytes([i] + [0] * 63) for i in range(256)]
lambda_2 = [bytes([0, i] + [0] * 62) for i in range(1, 256)]
query = b"".join(lambda_1 + lambda_2)
```

经过四轮扩散后，每个状态字节在全集上的异或和仍具有平衡性质。第五轮没有 `MixColumns`，因此可以逐字节猜测第五轮轮密钥，并在逆 S 盒之后检查这一平衡关系。

### 逐字节恢复末轮密钥

先根据 `ShiftPlanes` 的逆置换确定每个输出字节对应的猜测位置。对某个位置枚举 $k\in[0,255]$，对两组密文分别计算

$$
\bigoplus_{C\in\Lambda_j}S^{-1}(C_i\oplus k),\qquad j\in\{1,2\}.
$$

正确的末轮密钥字节会使两个异或和都等于零：

```python
for key_byte in range(256):
    xor_sum_1 = 0
    xor_sum_2 = 0

    for ct_1, ct_2 in zip(ciphertexts_1, ciphertexts_2):
        xor_sum_1 ^= inv_s_box[ct_1[pos] ^ key_byte]
        xor_sum_2 ^= inv_s_box[ct_2[pos] ^ key_byte]

    if xor_sum_1 == 0 and xor_sum_2 == 0:
        candidates[pos].append(key_byte)
```

对 64 个字节重复此过程。若个别位置仍有多个候选，枚举各位置候选的笛卡尔积并以 flag 格式验证即可。

### 逆转密钥扩展并解密

题目的 64 字节密钥扩展仍是 AES 风格：除每轮首字外，其余字都由本轮前一字与上一轮对应字异或得到。因此可从完整第五轮密钥逆向恢复前四轮，最终得到主密钥：

```python
master_key = bytes(inv_expand_key(last_round_key, 5))
plaintext = decrypt(master_key, encrypted_flag).rstrip(b"\x00")
print(plaintext)
```

官方求解脚本恢复出的明文为：

```text
UMDCTF{i_guess_it_should_be_called_a_cube_attack_in_this_case}
```

## 方法总结

- 核心技巧：对 5 轮大状态 AES 变体实施积分攻击，利用选择明文集合的平衡性质逐字节恢复末轮密钥。
- 识别信号：SPN 密码虽然分组和密钥很大，但轮数明显偏少、提供大量选择明文且末轮缺少线性混合时，应优先考虑积分攻击或 Square attack。
- 复用要点：第二个独立积分集合能显著压低错误候选；同时要先还原自定义状态布局和置换，否则对错误的密文字节做逆 S 盒会使平衡检查全部失败。
