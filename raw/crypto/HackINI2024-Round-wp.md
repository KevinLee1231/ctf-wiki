# round

## 题目简述

题目给出一段 AES-128-ECB 密文和某一轮的 16 字节 round key，但没有说明它来自第几轮。AES-128 密钥扩展是可逆的：只要枚举轮号并逆推主密钥，就能逐一尝试解密，用 flag 前缀识别正确结果。

## 解题过程

### 逆转 AES-128 key schedule

设某轮的四个 32 位字为 $W_i,W_{i+1},W_{i+2},W_{i+3}$。逆推上一组时，后三个字可依次由异或恢复，首字再使用 `RotWord`、`SubWord` 和对应 `Rcon` 还原。等价地，可以使用附件 requirements 中指定的 `aeskeyschedule` 实现。

官方 solver 将泄露值依次视作第 0 至第 9 轮 round key：

```python
from aeskeyschedule import reverse_key_schedule
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

enc = bytes.fromhex(
    "8fbabbaca2683abbb15d1340a7f79b4d"
    "5f26b687364f3af7e71809ee99fbd52b"
    "6875fa776b4a4624f775bc6d4f6d904a"
)
round_key = bytes.fromhex("8be4f2ecfb5522a1adfbdebd0be57d8d")

for round_number in range(10):
    master_key = reverse_key_schedule(round_key, round_number)
    plain = AES.new(master_key, AES.MODE_ECB).decrypt(enc)
    if b"shellmates{" in plain:
        print(round_number, master_key.hex(), unpad(plain, 16))
```

泄露的是第 7 轮 key，恢复出的 AES 主密钥为：

```text
58ad31dd1728253d0243bc290119a06f
```

最终 flag：

```text
shellmates{g3tt1ng_43s_K3y_fR0m_r0UnD_k3Y5}
```

仓库根 README 中的密文首块与官方 solver 不同，会解出多一个 `s` 的 `shellmatess{...}`；上述密文与官方 solver 和标准 flag 格式一致，因此采用该结果。

## 方法总结

- 核心技巧：利用 AES-128 key schedule 可逆的性质，从未知轮次的 round key 枚举恢复主密钥。
- 识别信号：泄露完整 16 字节轮密钥并不比泄露主密钥安全；轮号未知只增加很小的枚举空间。
- 复用要点：轮常量与轮号必须对应；每个候选都应实际解密并同时验证 PKCS#7 padding 和已知明文格式。
