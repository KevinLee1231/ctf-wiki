# Xor::Random

## 题目简述

Windows C++ 程序检查 flag 格式后，用伪随机字节对 30 字节密文交替异或。初始化函数固定执行 `srand(0x77)`；另一个候选 `srand(0x1919810)` 位于永远不会进入的分支，所以密钥可以通过复现 MSVCRT `rand()` 确定。

## 解题过程

主函数可简化为：

```c
srand(0x77);
v22 = rand();
v21 = rand();

for (int i = 0; i < 30; i++) {
    key = (i % 2 != 0) ? (v21 & 0xff) : ((v21 & 0xff) + 3);
    ciphertext[i] ^= key;
}
```

继续跟踪条件函数可以确认其返回值恒为 0，因此重新设置种子 `0x1919810` 的分支不会执行。动态调试也可以在 `v21` 最终赋值后读取其低字节。

MSVCRT 的 `rand()` 与 Linux glibc 实现不同，不能直接在 Linux Python/C 中调用 `rand` 代替。其状态更新可直接复现为：

$$
state=(214013\times state+2531011)\bmod2^{32}
$$

$$
rand=(state\gg16)\mathbin{\&}0x7fff
$$

完整解密脚本：

```python
import struct

cipher_qwords = [
    0x1221164E1F104F0C,
    0x171F240A4B10244B,
    0x1A2C5C2108074F09,
    99338668810000,
]

state = 0x77


def msvcrt_rand():
    global state
    state = (state * 214013 + 2531011) & 0xFFFFFFFF
    return (state >> 16) & 0x7FFF


msvcrt_rand()                 # v22
key = msvcrt_rand() & 0xFF    # v21 低字节，结果为 0x7b

ciphertext = b"".join(
    struct.pack("<Q", value)
    for value in cipher_qwords
)[:30]

plaintext = bytes(
    value ^ (key if index % 2 else key + 3)
    for index, value in enumerate(ciphertext)
)

print(plaintext.decode())
print(f"0xGame{{{plaintext.decode()}}}")
```

输出：

```text
r4nd0m_i5_n0t_alw4ys_'Random'!
0xGame{r4nd0m_i5_n0t_alw4ys_'Random'!}
```

## 方法总结

伪随机数可由种子和具体实现完全决定。本题先通过控制流确定实际种子，再按 Windows MSVCRT 算法复现第二次 `rand()`；密文常量还必须按小端序拆成字节，最后应用交替的 `key+3/key`。
