# Ezcpp

## 题目简述

程序由 C++ 编写，对 32 字节数据逐块执行类 TEA 加密。C++ 反编译产生的对象和类型转换使代码看起来较乱，但核心轮函数并没有复杂混淆。需要注意，这里使用了非标准常量 `0xdeadbeef`，且轮函数的两个移位都是左移，应严格按程序实现逆运算。

## 解题过程

### 识别分组与轮函数

关键函数每次接收两个 `uint32_t`，即一个 8 字节分组；密钥是四个 32 位整数，并迭代 32 轮。这些特征指向 TEA 结构。密钥与密文为：

```c
uint32_t key[4] = {1234, 2341, 3412, 4123};

unsigned char cipher[32] = {
    0x88, 0x04, 0xc6, 0x6a, 0x7f, 0xa7, 0xec, 0x27,
    0x6e, 0xbf, 0xb8, 0xaa, 0x0d, 0x3a, 0xad, 0xe7,
    0x7e, 0x52, 0xff, 0x8c, 0x8b, 0xef, 0x11, 0x9c,
    0x3d, 0xc3, 0xea, 0xfd, 0x23, 0x1f, 0x71, 0x4d
};
```

对比常见 TEA 模板时不能直接套库：本题的 `delta` 为 `0xdeadbeef`，而且常见 TEA 中的一个右移项在此处也是左移。反编译代码中的无符号溢出则是 $\bmod\ 2^{32}$ 运算的一部分。

### 实现解密

按加密的相反顺序，先从 `sum = delta * 32` 开始，每轮先撤销 `v1`，再撤销 `v0`：

```cpp
#include <cstdint>
#include <cstdio>

void decrypt(uint32_t *v, const uint32_t *k) {
    uint32_t v0 = v[0];
    uint32_t v1 = v[1];
    const uint32_t delta = 0xdeadbeef;
    uint32_t sum = delta * 32;

    for (int i = 0; i < 32; ++i) {
        v1 -= (v0 + sum)
            ^ (k[2] + (v0 << 4))
            ^ (k[3] + (v0 << 5));
        v0 -= (v1 + sum)
            ^ (k[0] + (v1 << 4))
            ^ (k[1] + (v1 << 5));
        sum -= delta;
    }

    v[0] = v0;
    v[1] = v1;
}

int main(void) {
    uint32_t key[4] = {1234, 2341, 3412, 4123};
    unsigned char cipher[33] = {
        0x88, 0x04, 0xc6, 0x6a, 0x7f, 0xa7, 0xec, 0x27,
        0x6e, 0xbf, 0xb8, 0xaa, 0x0d, 0x3a, 0xad, 0xe7,
        0x7e, 0x52, 0xff, 0x8c, 0x8b, 0xef, 0x11, 0x9c,
        0x3d, 0xc3, 0xea, 0xfd, 0x23, 0x1f, 0x71, 0x4d,
        0x00
    };

    decrypt(reinterpret_cast<uint32_t *>(&cipher[24]), key);
    decrypt(reinterpret_cast<uint32_t *>(&cipher[16]), key);
    decrypt(reinterpret_cast<uint32_t *>(&cipher[8]), key);
    decrypt(reinterpret_cast<uint32_t *>(&cipher[0]), key);

    std::printf("%s\n", cipher);
    return 0;
}
```

在题目所用的 x86/x86-64 小端环境中运行，可得：

```text
hgame{#Cpp_is_0bJ3cT_0r1enTeD?!}
```

官方题解还说明，题目第一天曾因密文后半段未成功加密而下线，第二天修复后重新上线；修复前后 flag 本身没有变化。

## 方法总结

- 识别 TEA 时应关注“两个 32 位数据字、四个 32 位密钥字和固定轮数”，但识别出家族不等于可以直接套标准实现。
- 逐项比对 `delta`、移位方向、加减顺序和密钥索引，任意一项不同都会使明文完全错误。
- C/C++ 中 `uint32_t` 的溢出自然实现了 $\bmod\ 2^{32}$；用 Python 重写时则要在每个关键步骤用 `& 0xffffffff` 截断。
- 密文按小端 `uint32_t` 解释；跨平台复现时最好显式指定字节序，避免依赖主机行为。
