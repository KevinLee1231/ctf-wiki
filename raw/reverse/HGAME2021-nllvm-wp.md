# nllvm

## 题目简述

程序通过修改编译器对全局变量做了额外加密，使静态反编译结果很长，但实际校验仍是标准 AES-256-CBC。根据 S-box、轮函数和 CBC 数据流识别算法后，从初始化后的全局区恢复 32 字节密钥、16 字节 IV 与 64 字节密文，直接调用标准 AES 实现即可解密，无需逐行逆向膨胀后的代码。

## 解题过程

官方源码片段暴露了关键全局量：

```c
uint8_t key[] = {"CryptoFAILUREforRSA2048Key!!!!!!"};

uint8_t iv[] = {
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
    0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f
};

uint8_t out[] = {
    0x91, 0xb3, 0xc1, 0xeb, 0x14, 0x5d, 0xd5, 0xce,
    0x3a, 0x1d, 0x30, 0xe4, 0x70, 0x6c, 0x6b, 0xd7,
    0x69, 0x78, 0x79, 0x02, 0xa3, 0xa5, 0xdf, 0x1b,
    0xfd, 0x1c, 0x02, 0x89, 0x14, 0x20, 0x7a, 0xfd,
    0x24, 0x52, 0xf8, 0xa9, 0xf9, 0xf1, 0x6b, 0x1c,
    0x0f, 0x5d, 0x50, 0x5b, 0xec, 0x42, 0xd1, 0x8c,
    0xb8, 0x12, 0xcf, 0x2c, 0xa9, 0x69, 0x31, 0x46,
    0xfd, 0x9b, 0xea, 0xde, 0xc8, 0xbf, 0x94, 0x69
};
```

字符串 `CryptoFAILUREforRSA2048Key!!!!!!` 恰好是 32 字节，说明使用 AES-256；IV 是连续的 `0x00` 到 `0x0f`，密文长度为 64 字节，满足 CBC 的 16 字节分组要求。反编译代码中如果还能看到 AES 逆 S-box、`xtime` 或轮密钥扩展，就足以排除“自定义密码”的误判。

直接解密：

```python
from Crypto.Cipher import AES


key = b"CryptoFAILUREforRSA2048Key!!!!!!"
iv = bytes(range(16))
ciphertext = bytes.fromhex(
    "91b3c1eb145dd5ce3a1d30e4706c6bd7"
    "69787902a3a5df1bfd1c028914207afd"
    "2452f8a9f9f16b1c0f5d505bec42d18c"
    "b812cf2ca9693146fd9beadec8bf9469"
)

assert len(key) == 32
assert len(iv) == 16
assert len(ciphertext) % 16 == 0

plaintext = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)
print(plaintext.decode())
```

输出为：

```text
hgame{cOsm0s_is_still_fight1ng_and_NEVER_GIVE_UP_O0o0o0oO00o00o}
```

解密结果正好占满 64 字节，没有 PKCS#7 填充，因此不能无条件调用 `unpad`；否则末字节 `}` 会被误判或直接触发异常。

## 方法总结

编译器级全局变量加密会显著增加静态代码量，却不一定改变底层算法。识别 AES 时应依靠 S-box、轮变换、密钥长度和分组模式等组合证据；确认标准实现后，优先在运行时或初始化完成后的内存中取出实际 key、IV、ciphertext，再用成熟库验证。还要检查明文是否真的带填充，不能机械套用 CBC 示例代码。
