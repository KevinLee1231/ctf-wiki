# L3akCTF 2024 Awesome Writeup

## 题目简述

页面只负责读取输入并调用 WebAssembly 模块导出的 `check` 函数。校验逻辑位于 `awesome_bg.wasm`：输入按小端序每 4 字节打包为一个 `u32`，不足偶数个整数时补零，再以两个 `u32` 为一组执行 32 轮 TEA，最后与 14 个常量比较。

决定性工作是从 WASM 中识别 TEA 的轮函数、密钥、密文和打包端序，而不是分析页面样式或 JavaScript 胶水代码。

## 解题过程

从 `awesome.js` 的导出表可以确认页面最终调用的是 `check`。用 `wasm2c`、`wasm2wat` 等工具还原该函数后，可整理出以下参数：

```python
enc = [
    2915842473, 3496841996,
    633173758, 1009180062,
    3608671705, 1697922677,
    2781256966, 1296367220,
    3020162604, 1282754354,
    3620747107, 79285426,
    3420268014, 1277316145,
]

key = [
    1416120629,
    2419151723,
    1702454895,
    1918125377,
]
```

轮常量是 TEA 标准使用的

$$
\delta=\mathtt{0x9e3779b9}.
$$

加密每轮先令 `sum += delta`，再依次更新 `v0`、`v1`：

```text
v0 += ((v1 << 4) + k0) ^ (v1 + sum) ^ ((v1 >> 5) + k1)
v1 += ((v0 << 4) + k2) ^ (v0 + sum) ^ ((v0 >> 5) + k3)
```

所以解密时应倒序更新 `v1`、`v0`，并在所有加减处截断到 32 位。完整脚本如下：

```python
import struct

MASK = 0xffffffff
DELTA = 0x9e3779b9

enc = [
    2915842473, 3496841996,
    633173758, 1009180062,
    3608671705, 1697922677,
    2781256966, 1296367220,
    3020162604, 1282754354,
    3620747107, 79285426,
    3420268014, 1277316145,
]

key = [
    1416120629,
    2419151723,
    1702454895,
    1918125377,
]


def decrypt_block(v0, v1):
    total = (DELTA * 32) & MASK

    for _ in range(32):
        v1 = (
            v1
            - (
                (((v0 << 4) & MASK) + key[2])
                ^ ((v0 + total) & MASK)
                ^ ((v0 >> 5) + key[3])
            )
        ) & MASK

        v0 = (
            v0
            - (
                (((v1 << 4) & MASK) + key[0])
                ^ ((v1 + total) & MASK)
                ^ ((v1 >> 5) + key[1])
            )
        ) & MASK

        total = (total - DELTA) & MASK

    return struct.pack("<II", v0, v1)


plain = b"".join(
    decrypt_block(enc[i], enc[i + 1])
    for i in range(0, len(enc), 2)
)

print(plain.rstrip(b"\x00").decode())
```

这里必须用 `<II` 按小端序还原，因为原程序的 `prepare_input` 将第 `i` 个字节左移 `8 * i` 位。运行得到：

```text
L3AK{Here's_YOur_w4sm_Challenge_n0t_th4t_hArd_right??}
```

## 方法总结

- WASM 题先从 JavaScript 导入导出关系定位真正入口，避免无目标地逆向整个模块。
- `0x9e3779b9`、32 轮、两个 32 位数据字和四个密钥字是识别 TEA 的强特征，但仍要核对具体更新顺序。
- Rust 的 `wrapping_add`、`wrapping_sub` 对应模 $2^{32}$ 运算，Python 实现必须在每一步使用 `& 0xffffffff`。
- 解密成功后若字符顺序异常，应优先检查输入打包端序，而不是随意翻转密文数组。
