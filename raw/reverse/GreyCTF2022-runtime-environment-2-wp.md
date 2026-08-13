# GreyCTF2022 - Runtime Environment 2

## 题目简述

第二题给出 Haskell 编码器和 32 字节密文。程序以当前 Unix 秒级时间为种子生成 xorshift32 序列，取每个状态的低 8 位作为一次性密码本；附件文件时间提供了足够窄的种子范围。

## 解题过程

从反编译或仓库源码还原更新函数：

```python
def xorshift32(x):
    x ^= (x << 13) & 0xffffffff
    x ^= x >> 17
    x ^= (x << 5) & 0xffffffff
    return x & 0xffffffff
```

读取 `encoded.bin` 的修改时间，将其附近若干秒作为候选种子。每生成一个新状态，就用 `state & 0xff` 与对应密文字节异或；以 `flag{` 前缀和可打印字符筛选。

```python
for seed in range(mtime - 120, mtime + 121):
    x = seed
    out = bytearray()
    for c in ciphertext:
        x = xorshift32(x)
        out.append(c ^ (x & 0xff))
    if out.startswith(b'flag{'):
        print(out.decode())
```

结果为：

```text
flag{Funct1on41_P4rad1s3_iZ_Fun}
```

## 方法总结

时间戳种子把密钥空间压缩到一个很小窗口，xorshift 又不是密码学安全 PRNG。恢复时要核对 Haskell `Int32` 的 32 位回绕、算术/逻辑右移以及“先更新还是先输出”；文件 mtime 只是中心值，枚举窗口应留出打包和复制产生的偏差。
