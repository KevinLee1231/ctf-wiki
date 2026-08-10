# Mystery

## 题目简述

程序的 `main` 几乎只有 `ptrace` 反调试，但运行时仍会要求输入 flag，说明关键逻辑位于初始化或退出阶段。检查 `.init_array`、`.fini_array` 及其引用后可以定位到构造函数 `sub_1220` 和析构函数 `sub_1100`：前者在 `main` 前改写密钥，后者在 `main` 后读取输入并校验。

算法分两层。第一层以 `keykey` 为标准 RC4 密钥，对字符串 `ban_debug!` 做 RC4，生成真正的 10 字节密钥；第二层用该密钥初始化 S 盒，再通过“加法版 RC4”解开 28 字节密文。

## 解题过程

在析构函数 `sub_1100` 中，输入先传给 `sub_1500`，结果再与固定密文比较，因此 `sub_1500` 是最终变换。它具有 RC4 的 KSA/PRGA 结构，但字节组合不是 XOR。若直接用静态字符串 `ban_debug!` 解密会失败，因为构造函数 `sub_1220` 已经调用标准 RC4 改写了这段内存。

先用标准 RC4 计算真实密钥：

```text
69 0d 5a b2 40 ea 19 3f 2f 6a
```

即十进制数组：

```text
105, 13, 90, 178, 64, 234, 25, 63, 47, 106
```

第二层的程序加密操作是逐字节减去 RC4 密钥流，因此解密时应逐字节加回，并按 `unsigned char` 对 256 取模。完整脚本如下：

```python
def rc4_xor(data, key):
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) % 256
        s[i], s[j] = s[j], s[i]

    i = j = 0
    output = bytearray()
    for value in data:
        i = (i + 1) % 256
        j = (j + s[i]) % 256
        s[i], s[j] = s[j], s[i]
        output.append(value ^ s[(s[i] + s[j]) % 256])
    return bytes(output)


stage_one_key = b"keykey"
mutable_key = b"ban_debug!"
real_key = rc4_xor(mutable_key, stage_one_key)
assert list(real_key) == [105, 13, 90, 178, 64, 234, 25, 63, 47, 106]

ciphertext = bytes([
    80, 66, 56, 77, 76, 84, 144, 111,
    254, 111, 188, 105, 185, 34, 124, 22,
    143, 68, 56, 74, 239, 55, 67, 192,
    162, 182, 52, 44,
])

s = list(range(256))
j = 0
for i in range(256):
    j = (j + s[i] + real_key[i % len(real_key)]) % 256
    s[i], s[j] = s[j], s[i]

i = j = 0
plaintext = bytearray()
for value in ciphertext:
    i = (i + 1) % 256
    j = (j + s[i]) % 256
    s[i], s[j] = s[j], s[i]
    stream_byte = s[(s[i] + s[j]) % 256]
    plaintext.append((value + stream_byte) & 0xFF)

print(plaintext.decode())
```

输出为：

```text
hgame{I826-2e904t-4t98-9i82}
```

[另一份参赛者复盘](https://rocketma.dev/2024/02/22/W3_remainder/)同样确认了构造/析构函数定位、`keykey`、`ban_debug!` 与两阶段 RC4 关系；正文已完整转写这些关键信息。

## 方法总结

- `main` 中没有业务逻辑但程序仍有明显行为时，应检查构造函数、析构函数、TLS callback、动态链接钩子等非主入口执行路径。
- 静态看到的密钥不一定是最终密钥；应追踪其全部写引用以及 `main` 前后的调用时序。
- 标准 RC4 使用 XOR，因而加解密相同；魔改为加减法后不再对称，必须确认加密方向并在解密时使用相反运算。
- 两层算法应分别验证中间值。本题先核对 10 字节真实密钥，再解最终密文，定位错误会更直接。
