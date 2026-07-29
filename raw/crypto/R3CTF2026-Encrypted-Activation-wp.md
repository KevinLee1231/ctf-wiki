# Encrypted Activation

## 题目简述

服务每轮随机选择一个 $0\le x<1024$ 的整数，将它拆成 5 个四进制数字并分别加密。玩家需要在密文上计算公开的 1024 项查找表：

$y=\operatorname{LUT}[x]$

再返回 $y$ 的 5 个四进制密文。连续完成 16 轮即可获得 flag。

题面强调“远端使用固定密钥，附件提供评估密钥”，容易让人直接实现完整的 FHE programmable bootstrap。但源码中的 `write_setup()` 明确把秘密密钥序列化为 `setup/client.bin`：

```python
sk, bsk, ksk = fhe.keygen()

client_blob = fhe.serialize_client_key(sk)
with open("setup/client.bin", "wb") as f:
    f.write(client_blob)
```

而服务的 `load_setup()` 又从同一文件恢复 `sk`。因此附件直接泄漏了远端固定秘密密钥。无需使用 bootstrap key 或 key-switch key；本地解密输入、查表、再正常加密输出即可。

## 解题过程

### 1. 理解四进制编码顺序

服务拆分整数时先取最低位：

```python
def extract_radix(v, s, n):
    d = []
    for _ in range(n):
        d.append(v % s)
        v //= s
    return d
```

所以密文列表依次对应：

$d_0,d_1,d_2,d_3,d_4$

且：

$x=d_0+4d_1+4^2d_2+4^3d_3+4^4d_4$

组合输出时使用相同约定：

```python
def combine_radix(d, s):
    v = 0
    for x in reversed(d):
        v = v * s + x
    return v
```

不能把收到的第一个符号误当成最高位。

### 2. 从 `client.bin` 恢复秘密密钥

序列化格式先保存两个维度，再直接保存两个二进制秘密向量的每个 64 位系数：

```python
def serialize_client_key(sk):
    out = bytearray()
    out += _u64le(Lvl0_n)
    out += _u64le(Lvl1_n)
    for coeff in sk.lvl0:
        out += _u64le(coeff)
    for coeff in sk.lvl1:
        out += _u64le(coeff)
    return bytes(out)
```

附件的 `parse_client_key()` 是其完整逆过程：

```python
with open("setup/client.bin", "rb") as f:
    sk = fhe.parse_client_key(f.read())
```

恢复出的密钥维度为：

```text
lvl0 = 622
lvl1 = 1024
```

服务端实际加密和解密输入使用 `lvl1`。本地可先做四个符号的往返自检：

```python
for digit in range(4):
    _, ct = fhe.encrypt_ciphertext(sk, digit)
    assert fhe.decrypt_ciphertext(sk, ct) == digit
```

### 3. 重建服务发送的 seeded LWE 密文

服务没有发送完整的 1024 项向量 $a$，而是发送：

```python
[
    (int.from_bytes(seed, "big"), ct.b),
    ...
]
```

`a` 由 32 字节种子通过 SHAKE-256 确定性展开：

```python
def decompress(seed):
    stream = hashlib.shake_256(
        seed + N.to_bytes(8, "little")
    ).digest(8 * N)
    return [
        int.from_bytes(stream[8*i:8*(i+1)], "little")
        for i in range(N)
    ]
```

JSON 中的整数会丢失种子开头的零字节，所以还原时必须指定固定长度：

```python
seed = seed_int.to_bytes(fhe.SEED_BYTES, "big")
a = fhe.decompress(seed)
ct = fhe.LWECiphertext(a=a, b=body)
digit = fhe.decrypt_ciphertext(sk, ct)
```

解密计算相位：

$\phi=b-\langle a,s\rangle\pmod{2^{64}}$

再按 $P=5$ 的编码间隔舍入。虽然有效消息范围是 $0$ 到 $4$，本题实际输入输出只使用四进制符号 $0$ 到 $3$。

### 4. 查表并重新加密输出

恢复五个输入符号后计算索引：

```python
x = combine_radix(in_digits, 4)
y = lut[x]
out_digits = extract_radix(y, 4, 5)
```

返回值不能伪造成明显的常量密文。服务会拒绝：

- `a` 中超过 16 项为 0；
- `a` 的所有元素完全相同；
- 向量长度不是 1024。

直接调用正常加密函数即可得到随机、格式合法且噪声足够小的密文：

```python
_, ct = fhe.encrypt_ciphertext(sk, digit)
blob = fhe.serialize_lwe_ciphertext(ct)
token = base64.b64encode(blob).decode()
```

服务只要求一行恰好包含 5 个 Base64 字符串，空格或逗号分隔均可。

### 5. 完整 solver

把以下脚本保存为附件目录中的 `solve.py`，确保同目录可导入 `fhe_core.py`，并能读取 `lut` 与 `setup/client.bin`：

```python
#!/usr/bin/env python3
import base64
import json
import socket
import sys

import fhe_core as fhe

S = 4
N_DIGITS = 5


def extract_radix(value, radix, count):
    digits = []
    for _ in range(count):
        digits.append(value % radix)
        value //= radix
    return digits


def combine_radix(digits, radix):
    value = 0
    for digit in reversed(digits):
        value = value * radix + digit
    return value


def recv_json(stream):
    while True:
        line = stream.readline()
        if not line:
            raise EOFError("server closed before sending a round")
        text = line.decode(errors="replace").strip()
        try:
            value = json.loads(text)
        except json.JSONDecodeError:
            print(text)
            continue
        if isinstance(value, dict) and "ciphertext" in value:
            return value


def main():
    if len(sys.argv) != 3:
        raise SystemExit(f"usage: {sys.argv[0]} HOST PORT")

    host = sys.argv[1]
    port = int(sys.argv[2])

    with open("setup/client.bin", "rb") as key_file:
        sk = fhe.parse_client_key(key_file.read())
    with open("lut", "r", encoding="utf-8") as lut_file:
        lut = [int(token) for token in lut_file.read().split()]

    if len(lut) != S ** N_DIGITS:
        raise ValueError("invalid LUT length")

    with socket.create_connection((host, port)) as sock:
        stream = sock.makefile("rwb", buffering=0)

        for expected_round in range(1, 17):
            payload = recv_json(stream)
            if payload.get("round") != expected_round:
                raise ValueError("unexpected round number")

            in_digits = []
            for seed_int, body in payload["ciphertext"]:
                seed = int(seed_int).to_bytes(fhe.SEED_BYTES, "big")
                a = fhe.decompress(seed)
                ct = fhe.LWECiphertext(a=a, b=int(body))
                in_digits.append(fhe.decrypt_ciphertext(sk, ct))

            x = combine_radix(in_digits, S)
            y = lut[x]
            out_digits = extract_radix(y, S, N_DIGITS)

            encoded = []
            for digit in out_digits:
                _, ct = fhe.encrypt_ciphertext(sk, digit)
                raw = fhe.serialize_lwe_ciphertext(ct)
                encoded.append(base64.b64encode(raw).decode())

            stream.write((" ".join(encoded) + "\n").encode())

        while True:
            line = stream.readline()
            if not line:
                break
            print(line.decode(errors="replace").rstrip())


if __name__ == "__main__":
    main()
```

运行方式：

```bash
cd attachment
python3 solve.py challenge-host 1336
```

每轮只做 5 次 LWE 解密和 5 次普通 LWE 加密，远低于 120 秒限制。

## 方法总结

本题预期表象是“对加密输入做 10 位 LUT 的同态求值”，但真正的安全边界在密钥分发：

1. `client.bin` 由 `serialize_client_key(sk)` 生成，包含完整的两个秘密向量；
2. 远端使用同一固定文件恢复秘密密钥；
3. seeded ciphertext 可由 JSON 中的 32 字节种子整数无损还原；
4. 解密 5 个四进制数字后直接查询公开 LUT；
5. 用泄漏的同一密钥正常加密输出，即可通过密文形态检查。

评估密钥可以公开，解密秘密密钥不能公开。文件名叫 `client.bin` 并不构成访问控制；审计密码协议附件时，应沿序列化函数确认每个文件实际包含的对象，而不是依据题面或文件名猜测其权限级别。
