# 维吉尼亚密码

## 题目简述

附件 `EZcrypto.txt` 是维吉尼亚密文。数字、花括号等非字母字符保持不变且不消耗密钥位置；利用已知 flag 前缀 `0xGame`，可以从对应字母的位移直接恢复循环密钥，而不是依靠猜测。

## 解题过程

密文为：

```text
0qKsfx{79ev4x0522t0e67s6x196ui52357w60u}
```

按字母位置对齐已知明文 `0xGame`：`x→q`、`G→K`、`a→s`、`m→f`，对应位移依次为 `t`、`e`、`s`、`t`，得到密钥 `test`。第五个字母 `e→x` 再次对应 `t`，也验证了四字符周期。

```python
ciphertext = "0qKsfx{79ev4x0522t0e67s6x196ui52357w60u}"
key = "test"

out = []
j = 0
for ch in ciphertext:
    if ch.isalpha():
        base = ord("A") if ch.isupper() else ord("a")
        shift = ord(key[j % len(key)]) - ord("a")
        out.append(chr((ord(ch) - base - shift) % 26 + base))
        j += 1
    else:
        out.append(ch)

print("".join(out))
```

解密结果为：

```text
0xGame{79ad4e0522a0a67a6e196be52357e60b}
```

## 方法总结

维吉尼亚密码是字母位移的周期重复，已知明文可直接给出对应密钥字符。恢复时必须确认题目如何处理大小写、非字母字符以及密钥索引；原稿中把密钥写成 `Game` 与密文不符，本题实际密钥是 `test`。
