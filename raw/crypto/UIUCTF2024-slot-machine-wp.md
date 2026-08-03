# Slot Machine

## 题目简述

服务允许提交任意十六进制“幸运数字”，按字节反转后计算一次 SHA-256，再把摘要反转成显示顺序。若摘要前 `length` 个十六进制字符完全相同，就返回 Flag 的前 `length` 个字符，最多 32 个。Flag 长 24 字符，因此需要一个 SHA-256 输出至少有 24 个相同前导半字节的已知输入。

## 解题过程

从零搜索 96 位相同前缀不可行，但比特币工作量证明已经公开提供了大量“双 SHA-256 后带长零前缀”的区块头。选择高度 756951 的区块，其 80 字节头按网络序列化字段拼接为：

```text
version:     00004020
prev_block:  54918D671610FC65C9ACB5DDB46D30E1C25194DAA00D05000000000000000000
merkle_root: 5A0058B0C79451FF7883A5804AF798BA2C55BB62533E46B7396EDFFA1E6FC462
timestamp:   2D8C3B63
bits:        AEF90817
nonce:       8C0F23C1
```

区块头第一次 SHA-256 的内部摘要为：

```text
93CCD10E30712E566E0BC0189C791E609B11FC17190B00EB50D6FA8B4909B2F5
```

把这段值作为幸运数字提交，服务的 `bytes.fromhex(...)[::-1]` 恰好恢复再次哈希所需的字节序；第二次 SHA-256 并反转显示后为：

```text
0000000000000000000000005D6F06154C8685146AA7BC3DC9843876C9CEFD0F
```

其前 24 个十六进制字符全为 `0`。本地可按服务逻辑复核：

```python
from hashlib import sha256

lucky = "93CCD10E30712E566E0BC0189C791E609B11FC17190B00EB50D6FA8B4909B2F5"
shown_hash = sha256(bytes.fromhex(lucky)[::-1]).digest()[::-1].hex()
assert shown_hash.startswith("0" * 24)
print(shown_hash)
```

随后把槽位数设为 `24`，一次取回完整 Flag：

```text
uiuctf{keep_going!_3cyd}
```

## 方法总结

- 题目并不要求寻找指定消息的 SHA-256 原像，只要求任意已知输入具有重复前缀；公开的工作量证明区块可以直接复用。
- 比特币区块哈希是对 80 字节头做两次 SHA-256，并在通常展示时反转字节序。本题额外的输入、输出反转正好模拟了第二轮与展示过程。
- 槽位数同时控制判定长度和 Flag 泄漏长度，必须至少覆盖完整 Flag；提交前应严格按服务代码验证字节序与前导零数量。
