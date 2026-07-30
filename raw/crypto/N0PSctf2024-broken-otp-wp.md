# Broken OTP

## 题目简述

服务提供两个互斥操作：加密任意消息，或返回加密后的 secret。每次连接只执行一次操作。加密本身是明文与伪随机字节流异或：

```python
def kg(length):
    return bytes(random.randint(0, 255) for _ in range(length))

def encrypt(plain):
    key = kg(len(plain))
    return bytes(k ^ p for k, p in zip(key, plain)).hex()
```

源码前半段用 `eval`、`getattr`、`dir` 和十六进制字符串混淆了随机种子的生成；题面特意给出 Python 3.12.3，因为 `dir(__builtins__)` 的索引解析依赖该版本。还原后核心逻辑为：

```python
random.seed(round(time.time()) + int.from_bytes(secret.encode()))
```

secret 固定不变，时间被四舍五入到秒，因此同一秒启动的两个服务进程会生成相同的密钥流。

## 解题过程

### 还原混淆

关键变量可化简为：

- `e = eval`
- `g = getattr`
- `b = bytearray`
- `i = int`
- `d(x) = bytearray.fromhex(x).decode()`
- `fb = int.from_bytes`
- `r = random.seed`

所以种子不是未知的纯随机值，而是“固定 secret 对应的大整数 + 当前秒”。攻击无需恢复该种子；只需让两个连接落在同一秒。

### 复用密钥流

设已知明文为 $P$，secret 为 $S$，同一伪随机密钥流为 $K$，两个密文分别为：

$$
C_P=P\oplus K,\qquad C_S=S\oplus K
$$

两式异或并再次异或 $P$：

$$
S=C_P\oplus C_S\oplus P
$$

为缩短两个进程初始化的时间差，先建立两条连接，再分别选择“加密消息”和“加密 secret”。以下脚本保留了完整攻击逻辑，地址由复现环境传入：

```python
from pwn import remote


def xor3(a: bytes, b: bytes, c: bytes) -> bytes:
    return bytes(x ^ y ^ z for x, y, z in zip(a, b, c))


def solve(host: str, port: int) -> bytes:
    known = b"A" * 19

    left = remote(host, port)
    right = remote(host, port)

    left.sendlineafter(b"Enter your choice: ", b"1")
    right.sendlineafter(b"Enter your choice: ", b"2")

    left.sendlineafter(b"Please enter the message you wish to encrypt: ", known)
    known_ct = bytes.fromhex(
        left.recvline_contains(b"Your encrypted message is: ")
        .split(b": ", 1)[1]
        .decode()
    )
    secret_ct = bytes.fromhex(
        right.recvline_contains(b"The secret is: ")
        .split(b": ", 1)[1]
        .decode()
    )

    left.close()
    right.close()
    return xor3(known_ct, secret_ct, known)


# print(solve(HOST, PORT))
```

若两个进程恰好跨秒，结果不会以 `N0PS{` 开头，重新建立两条连接即可。成功恢复：

```text
N0PS{0tP_k3Y_r3u53}
```

## 方法总结

- 核心技巧：利用秒级时间种子造成的伪随机密钥流复用，通过已知明文消去密钥流。
- 识别信号：异或“OTP”由 `random` 生成、种子含低精度时间、每个连接重新初始化且 secret 固定。
- 复用要点：未知种子不一定需要恢复；若能让两个会话获得同一密钥流，直接利用 $C_1\oplus C_2=P_1\oplus P_2$ 通常更可靠。跨秒是主要不稳定因素，应并行建连并对输出格式做重试判断。
