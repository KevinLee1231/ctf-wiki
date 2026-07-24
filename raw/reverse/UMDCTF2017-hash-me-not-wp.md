# Hash Me Not

## 题目简述

`HashMeNot` 对验证码逐字符处理。每个字符先被转换成由 `U`、`M`、`D` 组成的自定义三进制表示，再计算 CRC32，最后把所有 8 位十六进制摘要直接拼接。CRC32 不可直接求逆，但输入域只是可打印单字节，完全可以建立反查表。

## 解题过程

反编译得到单字符变换：

1. 反复取 `value % 3`，余数 0、1、2 分别映射为 `U`、`M`、`D`；
2. 更新 `value = (value + 1) // 3`；
3. 逆序拼接这些字符；
4. 对结果计算 CRC32，并用 8 位十六进制表示。

把二进制中的目标串按每 8 字符切分，再枚举 `0x20` 到 `0x7e`：

```python
import zlib

target = (
    "b9850b343b9e936db9850b34c5d42e20431b4157975467ae"
    "e4d47935b3d7255a31ccbd038438fff8c483ccef80ec08d6"
    "4cca7ad8eb0542ba"
)

def transform(value):
    digits = []
    while value:
        digits.append("UMD"[value % 3])
        value = (value + 1) // 3
    encoded = "".join(reversed(digits)).encode()
    return f"{zlib.crc32(encoded):08x}"

lookup = {transform(value): chr(value) for value in range(0x20, 0x7F)}
chunks = [target[index:index + 8] for index in range(0, len(target), 8)]
print("".join(lookup[chunk] for chunk in chunks))
```

每个摘要块在可打印 ASCII 范围内都只有一个原像，输出：

```text
lol@w3aKH4Shes
```

最终 flag：

```text
UMDCTF-{lol@w3aKH4Shes}
```

其 SHA-256 与 README 摘要一致。

## 方法总结

哈希或校验和“不可逆”不等于无法恢复：若每次只处理一个字符，输入空间只有 95 个候选，预计算反查表就足够。判断攻击成本时必须看分块方式和输入熵，而不能只看函数名称。
