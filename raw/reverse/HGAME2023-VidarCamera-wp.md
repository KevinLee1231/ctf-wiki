# VidarCamera

## 题目简述

附件是使用 Kotlin 编写的 Android 应用。程序要求输入 40 字符序列号，将其按小端序打包成 10 个 32 位整数，再用一套修改过的 XTEA 变体加密并与常量数组比较。

## 解题过程

在反编译结果中定位输入长度判断和常量数组，可确认算法与标准 XTEA 有四点差异：

- `delta` 改为 `0x34566543`；
- 更新 `v0` 时额外异或当前的 `sum`；
- `do ... while (round <= 32)` 实际执行 33 轮；
- 10 个字不是按 `(0,1)、(2,3)` 分组，而是依次加密重叠窗口 `(0,1)、(1,2)、...、(8,9)`。

密文和密钥为：

```text
ciphertext = [
    637666042, 457511012, -2038734351, 578827205, -245529892,
    -1652281167, 435335655, 733644188, 705177885, -596608744,
]
key = [2233, 4455, 6677, 8899]
```

由于加密窗口互相重叠，解密必须从最后一对 `(8,9)` 倒序处理到 `(0,1)`：

```python
from ctypes import c_uint32


def decrypt_pair(left: int, right: int, key: list[int]) -> tuple[int, int]:
    v0 = c_uint32(left)
    v1 = c_uint32(right)
    delta = 0x34566543
    total = c_uint32(delta * 33)

    for _ in range(33):
        total.value -= delta
        v1.value -= (
            (((v0.value << 4) ^ (v0.value >> 5)) + v0.value)
            ^ (total.value + key[(total.value >> 11) & 3])
        )
        v0.value -= (
            (((v1.value << 4) ^ (v1.value >> 5)) + v1.value)
            ^ (total.value + key[total.value & 3])
            ^ total.value
        )

    return v0.value, v1.value


words = [
    637666042, 457511012, -2038734351, 578827205, -245529892,
    -1652281167, 435335655, 733644188, 705177885, -596608744,
]
key = [2233, 4455, 6677, 8899]

for index in range(len(words) - 2, -1, -1):
    words[index], words[index + 1] = decrypt_pair(
        words[index], words[index + 1], key
    )

flag = "".join(word.to_bytes(4, "little").decode() for word in words)
print(flag)
```

运行得到：

```text
hgame{d8c1d7d34573434ea8dfe5db40fbb25c0}
```

该结果已用脚本实际解密验证。PDF 中的 Kotlin 反编译截图已还原为算法差异、常量和完整求解代码，没有保留纯代码截图。

## 方法总结

识别出“类似 XTEA”只是起点，真正决定结果的是实现差异。应逐项核对常量、轮数、运算顺序、32 位溢出和分组方式。本题尤其容易把闭区间的 33 轮误看成 32 轮，或忽略重叠窗口；解密时必须同时逆转单轮操作和窗口处理顺序。
