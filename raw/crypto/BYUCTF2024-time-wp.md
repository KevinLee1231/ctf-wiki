# Time

## 题目简述

程序读取 flag 后执行 `srand(time(NULL))`，再逐字节与 `rand() % 256` 异或并输出整数。虽然题目位于官方 Rev 目录，真正的弱点是可预测的时间种子伪随机数，因此按 Crypto 归档。

## 解题过程

C/C++ 的 `rand()` 并不是密码学随机数。服务把 Unix 时间戳精确到秒作为种子；在接收密文的同时记录本地时间，就只需枚举网络与容器时钟造成的几秒偏差。需要使用与服务相同 libc 的 `srand/rand` 序列，而不能直接替换成 Python 的 `random`：

```python
from ctypes import CDLL
from time import time

libc = CDLL("libc.so.6")
enc = [...]  # 服务输出的整数
now = int(time())

for delta in range(-10, 11):
    libc.srand(now + delta)
    plain = bytes(v ^ (libc.rand() % 256) for v in enc)
    if plain.startswith(b"byuctf{"):
        print(delta, plain.decode())
```

正确种子得到：

```text
byuctf{ooooooooh_a_seeded_PRNGGGGGGGGGG}
```

## 方法总结

时间戳只缩小了随机序列的搜索空间，并没有提供秘密性。复现此类 PRNG 时必须匹配具体实现和调用次数；实际加密应使用操作系统 CSPRNG，并避免把可预测随机流直接当作一次一密密钥流。
