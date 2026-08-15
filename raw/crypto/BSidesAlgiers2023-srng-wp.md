# SRNG

## 题目简述

服务实现了一个名为 `Spooder` 的自制伪随机数生成器。内部状态由 `rand` 和递增计数器 `i` 组成，每次调用执行：

```python
self.rand = pow(self.i, self.rand, modulus)
self.i += 1
```

欢迎信息会连续泄露一组随机数、一个随机字符串，以及“随机填充 + flag 异或密文”。虽然初始状态由时间戳生成，但在加密 flag 前，状态已经被限制在一个很小的模数范围内，可以直接穷举。

## 解题过程

初始化函数先调用一次 `generate_random()`，因此生成欢迎信息前 `i=3`。欢迎信息按从左到右的顺序继续消耗状态：

1. 生成随机数列表时，先调用一次决定列表长度，再为每个元素调用一次。
2. 生成示例随机字符串时，先调用一次决定长度，再为每个字符调用一次。
3. 随后才调用 `spooder_encryption(FLAG)`。

因此只要统计列表和字符串的实际长度，就能得到 flag 加密开始前的准确 `i`：

```python
i = 3
i += 1 + len(leaked_numbers)
i += 1 + len(leaked_padding)
```

示例随机字符串的每个字符都由 `generate_random(0xd7fb)` 产生，所以此时的 `rand` 必在 `0` 到 `0xd7fa` 之间。对这 $0xd7fb$ 种状态逐一重放加密过程即可：

```python
class Spooder:
    def __init__(self):
        self.rand = 0
        self.i = 0

    def generate_random(self, modulus=0x10FFFF):
        self.rand = pow(self.i, self.rand, modulus)
        self.i += 1
        return self.rand

    def generate_padding(self, length_modulus=0x101):
        length = self.generate_random(length_modulus)
        return "".join(
            chr(self.generate_random(0xD7FB)) for _ in range(length)
        )


def recover(cipher_hex, state_i):
    cipher = bytes.fromhex(cipher_hex).decode()

    for state_rand in range(0xD7FB):
        rng = Spooder()
        rng.i = state_i
        rng.rand = state_rand

        padding = rng.generate_padding()
        encrypted_flag = cipher[len(padding):]
        candidate = "".join(
            chr(ord(ch) ^ rng.generate_random(0xD7FB))
            for ch in encrypted_flag
        )
        if candidate.startswith("shellmates{"):
            return candidate

    raise ValueError("flag not found")
```

把服务泄露的十六进制密文和统计出的 `i` 传给 `recover`，得到：

```text
shellmates{5p00d3R_Fl4g_f0r_sPooDeR_cH4lL3nge}
```

## 方法总结

自制生成器即使初始种子来自时间戳，也可能在后续运算中丢失大量状态空间。本题先用小模数输出覆盖了原状态，再把同一状态用于生成填充和异或密钥流；结合可观察的调用次数，攻击者只需穷举不到 $2^{16}$ 个状态。最终 flag 为 `shellmates{5p00d3R_Fl4g_f0r_sPooDeR_cH4lL3nge}`。
