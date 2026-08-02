# seeds

## 题目简述

服务使用自定义线性同余生成器产生 AES-256-ECB 密钥。生成器未显式传入 seed 时直接取 `time.asctime()`，把该可读时间字符串转成整数；服务启动后立即用第一个 32 字节输出加密 flag 并打印密文。题目提示服务器位于美国东部时区，因此密钥只取决于连接时刻附近、按该时区格式化的秒级时间字符串。

## 解题过程

生成器状态更新公式为

$$s_{i+1}=(157s_i+1)\bmod2^{32}.$$

`randbytes(32)` 连续拼接 8 个 32 位状态，恰好构成 AES-256 密钥。记录收到初始密文时的本地时间，把附近若干秒转换为 `America/New_York` 时区的 `asctime` 字符串，逐个重建密钥并解密；以 `tjctf{` 和合法 PKCS#7 填充作为成功条件。这样比官方只尝试单个当前秒的脚本更能容忍网络延迟和秒边界。

```python
import ast
import time
from datetime import datetime
from zoneinfo import ZoneInfo
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

class RandomGenerator:
    def __init__(self, seed: str):
        self.state = int.from_bytes(seed.encode(), "big")

    def next32(self) -> int:
        self.state = (157 * self.state + 1) % (2 ** 32)
        return self.state

    def randbytes(self, length: int) -> bytes:
        out = b""
        while len(out) < length:
            out += self.next32().to_bytes(4, "big")
        return out[:length]

# 收到服务的 `ciphertext = b'...'` 行时立即记录时间。
line = input("粘贴服务输出的 ciphertext 行：").strip()
received_at = time.time()
ciphertext_literal = line.split("=", 1)[1].strip()
ciphertext = ast.literal_eval(ciphertext_literal)

for delta in range(-10, 11):
    eastern = datetime.fromtimestamp(
        received_at + delta, ZoneInfo("America/New_York")
    )
    seed = time.asctime(eastern.timetuple())
    key = RandomGenerator(seed).randbytes(32)
    padded = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
    try:
        plaintext = unpad(padded, 16)
    except ValueError:
        continue
    if plaintext.startswith(b"tjctf{"):
        print(plaintext.decode())
        break
```

仓库中的 flag 与官方求解脚本对应结果为：

```text
tjctf{h4rv3st_t1me}
```

## 方法总结

- 核心技巧：枚举秒级时间 seed，重建 LCG 输出并派生 AES 密钥。
- 识别信号：`time.asctime()` 直接作为 seed、明确的服务器时区提示，以及连接后立即给出的密文。
- 复用要点：不要只猜一个时间点；应覆盖网络延迟窗口、夏令时和时区差异，并用明文格式与 padding 同时验证候选。
