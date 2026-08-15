# Death Star Trial Version

## 题目简述

程序伪装成许可证签名验证器：它加载一段 DER 公钥，读取用户输入后调用 OpenSSL 验签，看起来必须伪造签名才能通过。真正的校验却藏在失败路径的 `cleanup_crypto_resources` 中。该函数把许可证逐字节与公钥材料第 16 字节之后的 16 字节循环异或，再与固定的 47 字节 `checksum_data` 比较。因此，本题的核心不是攻击公钥密码，而是识别“清理函数”中的隐藏验证逻辑并直接逆运算。

## 解题过程

### 跟踪实际控制流

`main` 对许可证调用 `verify_license_signature`。签名数据由程序内部生成，并不能由用户一并提交；验签失败后并非立即退出，而是把原始许可证传给清理函数：

```c
int verify_result = verify_license_signature(license);
if (verify_result != 1) {
    cleanup_crypto_resources(license);
    return verify_result;
}
```

因此应继续分析 `cleanup_crypto_resources`，不能把 OpenSSL 验签当成唯一目标。该函数的有效逻辑可概括为：

```c
const unsigned char *key_data = &static_key_material[16];

for (size_t i = 0; i < license_len; i++) {
    buffer[i] = license[i] ^ key_data[i % 16];
}

if (license_len == sizeof(checksum_data) &&
    memcmp(buffer, checksum_data, sizeof(checksum_data)) == 0) {
    /* valid license */
}
```

这同时给出了长度约束：正确输入必须恰好为 47 字节。

### 逆转循环异或

异或是自身的逆运算。若目标字节为 $c_i$、循环密钥为 $k_i$，则许可证字节为：

$$
p_i = c_i \oplus k_{i \bmod 16}
$$

密钥并不是 DER 公钥的开头，而是 `static_key_material[16:32]`：

```python
key = bytes([
    0x01, 0x05, 0x00, 0x03, 0x82, 0x01, 0x0f, 0x00,
    0x30, 0x82, 0x01, 0x0a, 0x02, 0x82, 0x01, 0x01,
])

checksum = bytes([
    0x72, 0x6d, 0x65, 0x6f, 0xee, 0x6c, 0x6e, 0x74,
    0x55, 0xf1, 0x7a, 0x62, 0x33, 0xe6, 0x65, 0x32,
    0x6f, 0x5a, 0x31, 0x6d, 0xdd, 0x71, 0x63, 0x34,
    0x01, 0xec, 0x5e, 0x79, 0x33, 0xe5, 0x69, 0x75,
    0x5e, 0x66, 0x6c, 0x30, 0xb6, 0x6f, 0x7a, 0x70,
    0x6f, 0xee, 0x31, 0x6d, 0x33, 0xe1, 0x7c,
])

license_text = bytes(
    value ^ key[i % len(key)]
    for i, value in enumerate(checksum)
)
print(license_text.decode())
```

得到：

```text
shellmates{h1dd3n_1n_pl41n_s1ght_cl34nup_l0g1c}
```

## 方法总结

这道题利用了“名称与职责不一致”的逆向陷阱：真正的成功条件位于失败分支和清理函数，而复杂的公钥验签只是干扰。分析此类程序时应沿所有退出路径继续追踪副作用，并重点检查固定长度比较、循环异或和嵌入常量。确定关系为 `input XOR key == checksum` 后，直接用相同异或即可恢复输入，无须攻击签名算法。
