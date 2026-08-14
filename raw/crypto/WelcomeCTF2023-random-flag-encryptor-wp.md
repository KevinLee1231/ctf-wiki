# random flag encryptor

## 题目简述

附件中的 Python 程序以当前 Unix 时间戳 $t$ 经过若干移位和异或后作为 `random.seed`，再为 Flag 的每个字符生成 1 至 255 的伪随机字节并异或。密文写入 `encrypted_flag.bin`。

Python `random` 不是密码学安全随机源；更关键的是种子只依赖秒级时间。已知比赛开始时间后，搜索空间只是附近的一段时间窗口。

## 解题过程

种子函数为：

```python
def make_seed(t):
    return (
        t
        ^ ((t & 0xFFFF) * (t >> 16))
        ^ ((t << 5) & 0xFFFFFFFF)
        ^ ((t >> 3) & 0xFFFFFFFF)
        ^ ((t << 4) & 0xFFFFFFFF)
    ) & 0xFFFFFFFF
```

从比赛开始时间附近逐秒尝试。每个候选时间都重新初始化 PRNG，生成与密文等长的密钥流并异或；出现 `greyhats{` 即视为强校验：

```python
import random
import time

ct = open("encrypted_flag.bin", "rb").read()
t = int(time.mktime((2023, 8, 25, 22, 0, 0, 0, 0, 0)))

while True:
    random.seed(make_seed(t))
    pt = bytes(c ^ random.randint(1, 255) for c in ct)
    if pt.startswith(b"greyhats{"):
        print(pt.decode())
        break
    t -= 1
```

正确时间对应 2023-08-25 21:47:23（GMT+8），恢复出：

```text
greyhats{1_w15h_7H3r3_W45_4_r4nd0m_cH4ll3n93_93n3r470r}
```

## 方法总结

- 核心技巧：围绕已知事件时间爆破秒级时间种子，重放 Python PRNG 密钥流。
- 识别信号：`random.seed(int(time.time()))` 或时间戳经可逆/低熵变换后直接用于加密。
- 复用要点：搜索时要确认时区和时间方向，并用固定格式、可打印字符或校验码快速排除错误种子。
