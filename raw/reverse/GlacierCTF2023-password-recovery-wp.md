# GlacierCTF2023 - password_recovery

## 题目简述

程序读取用户名和密码，用固定初始状态 `0x1337` 的 xorshift 产生交换下标，对用户名原地置换；随后逐字节与 64 位常量 `0x1337DEADBEEFC0DE` 的小端字节循环异或，再将结果压到 `a` 至 `z`，最后用 `strcmp` 比较。

## 解题过程

算法没有秘密随机量，完整复现即可。C 中的 `size_t` 是无符号 64 位，xorshift 每一步都按 $2^{64}$ 截断；密钥在 x86-64 上按小端字节序使用。

```python
MASK = (1 << 64) - 1

def xorshift(x):
    x = (x ^ ((x << 13) & MASK)) & MASK
    x = (x ^ (x >> 17)) & MASK
    x = (x ^ ((x << 5) & MASK)) & MASK
    return x

def password_for(username):
    state = 0x1337
    name = list(username.encode())
    for i in range(len(name)):
        state = xorshift(state)
        j = state % len(name)
        name[i], name[j] = name[j], name[i]

    key = (0x1337DEADBEEFC0DE).to_bytes(8, "little")
    return bytes(
        ((value ^ key[i % 8]) % 26) + ord("a")
        for i, value in enumerate(name)
    ).decode()

print(password_for("LosCapitan"))
```

当前源码对 `LosCapitan` 生成 `stmhrrlcjy`。也可以在动态调试时断在 `strcmp`，直接读取第一个参数指向的变换后用户名。仓库没有保存该题的赛事 flag 配置，只保留了校验程序和通过条件，因此不虚构最终 flag。

## 方法总结

固定 PRNG、固定密钥和确定性置换都不构成密码保护。复现此类 checker 时，最容易出错的是整数溢出、端序和原地交换顺序；若只要求快速通过，比较函数断点则能直接观察期望值。
