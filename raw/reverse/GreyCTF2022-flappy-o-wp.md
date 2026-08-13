# GreyCTF2022 - flappy-o

## 题目简述

原生版把 flag 与线性反馈移位寄存器生成的字节流异或。LFSR 参数和种子均硬编码在程序中，因此加密流可完全重放。

## 解题过程

第一版使用 16 位状态，初始值为 `0xabcd`，反馈多项式常量为 `0x82EE`。从反编译结果确认每轮移位方向、反馈条件，以及一个 flag 字节消耗多少个 LFSR 输出位；按相同顺序生成密钥字节，与静态数组异或。

```python
state = 0xabcd

def step():
    global state
    out = state & 1
    state >>= 1
    if out:
        state ^= 0x82EE
    return out

def key_byte():
    return sum(step() << i for i in range(8))

plain = bytes(c ^ key_byte() for c in encrypted_flag)
```

若反编译显示按高位拼接，只需相应调整位序；用已知前缀 `grey{` 可快速校验。正确结果为：

```text
grey{y0u_4r3_pr0_4t_7h1s_g4m3_b6d8745a1cc8d51effb86690bf4b27c9}
```

## 方法总结

已知种子和反馈多项式的 LFSR 只提供可预测伪随机流，不能当作密钥流生成器。复现时最容易错的是移位方向、输出位取自高端还是低端、以及字节拼接端序；先解前五个字节验证能节省大量排错时间。
