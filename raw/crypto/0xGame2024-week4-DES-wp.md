# DES

## 题目简述

题目保留了 DES 的扩展置换、8 个 S 盒和 P 置换，但整体并不是标准 DES：没有初始/末尾置换，只执行两轮 Feistel，并且两轮直接复用同一个 48 位密钥。服务总共允许 11 次操作，提交完整 6 字节密钥即可获得 flag。

由于轮数只有两轮，可以从选择明文及其密文直接算出两次轮函数的输入和输出，再把 48 位密钥拆成 8 组独立的 6 位候选进行枚举，不需要搜索完整密钥空间，也不必构造复杂差分路径。

## 解题过程

把明文记为 $(L_0,R_0)$。两轮 Feistel 满足

$$
L_1=R_0,\qquad R_1=L_0\oplus F(R_0,K),
$$

$$
L_2=R_1,\qquad R_2=R_0\oplus F(R_1,K).
$$

源码最终返回 `R + L`，所以密文左右两半实际为 $(R_2,L_2)=(R_2,R_1)$。已知一组明密文后可以直接得到

$$
F(R_0,K)=L_0\oplus R_1,
$$

$$
F(R_1,K)=R_0\oplus R_2.
$$

对轮函数输出逆 P 置换后，就得到 8 个 S 盒各自的 4 位输出。第 $i$ 个 S 盒只依赖扩展结果对应的 6 位和密钥的第 $i$ 个 6 位分组，因此每组只需枚举 $2^6=64$ 种。多取几组明密文求交即可唯一确定全部 8 组。

下面脚本使用 5 次加密和 1 次密钥提交，未超过 11 次限制，并直接复用附件中的 `Util.py`：

```python
from pwn import remote

import Util as U


HOST = "TARGET"
PORT = 10008

P_TABLE = [
    15, 6, 19, 20, 28, 11, 27, 16,
    0, 14, 22, 25, 4, 17, 30, 9,
    1, 7, 23, 13, 31, 26, 2, 8,
    18, 12, 29, 5, 21, 10, 3, 24,
]

PLAINTEXTS = [
    bytes.fromhex("0000000000000000"),
    bytes.fromhex("0123456789abcdef"),
    bytes.fromhex("fedcba9876543210"),
    bytes.fromhex("0011223344556677"),
    bytes.fromhex("89abcdef01234567"),
]


def xor_bits(a, b):
    return [x ^ y for x, y in zip(a, b)]


def inverse_p(output):
    before_p = [0] * 32
    for output_index, input_index in enumerate(P_TABLE):
        before_p[input_index] = output[output_index]
    return before_p


def encrypt_oracle(io, plaintext):
    io.sendlineafter(b">", b"E")
    io.sendlineafter(b">", plaintext.hex().encode())
    io.recvuntil(b"result:")
    return bytes.fromhex(io.recvline().strip().decode())


def add_constraints(plaintext, ciphertext, constraints):
    plain_bits = U.bytes2bin(plaintext)
    cipher_bits = U.bytes2bin(ciphertext)

    l0, r0 = plain_bits[:32], plain_bits[32:]
    r2, r1 = cipher_bits[:32], cipher_bits[32:]

    constraints.append((r0, xor_bits(l0, r1)))
    constraints.append((r1, xor_bits(r0, r2)))


def matches(sbox_index, key_chunk, r_value, f_output):
    expanded = U.EP(r_value)
    key_bits = [int(bit) for bit in f"{key_chunk:06b}"]

    # U.S_box 同时处理 8 组输入；这里只比较目标 S 盒的 4 位输出。
    sbox_input = [0] * 48
    begin = 6 * sbox_index
    sbox_input[begin:begin + 6] = xor_bits(
        expanded[begin:begin + 6], key_bits
    )
    got = U.S_box(sbox_input)[4 * sbox_index:4 * sbox_index + 4]

    expected = inverse_p(f_output)[
        4 * sbox_index:4 * sbox_index + 4
    ]
    return got == expected


io = remote(HOST, PORT)
constraints = []

for plaintext in PLAINTEXTS:
    ciphertext = encrypt_oracle(io, plaintext)
    add_constraints(plaintext, ciphertext, constraints)

chunks = []
for sbox_index in range(8):
    candidates = [
        key_chunk
        for key_chunk in range(64)
        if all(
            matches(sbox_index, key_chunk, r_value, f_output)
            for r_value, f_output in constraints
        )
    ]
    assert len(candidates) == 1, (sbox_index, candidates)
    chunks.append(candidates[0])

key_bits = "".join(f"{chunk:06b}" for chunk in chunks)
key = int(key_bits, 2).to_bytes(6, "big")

io.sendlineafter(b">", b"F")
io.sendlineafter(b">", key.hex().encode())
print(io.recvline().decode().strip())
```

恢复密钥并提交后得到：

```text
0xGame{de22c1fb-7f6b-42b1-a19a-5c16e12c70a4}
```

`secret.py` 中 flag 后面的 `\x04\x04\x04\x04` 是填充字节，不属于 flag 正文。

## 方法总结

面对“简化 DES”应先按源码重建轮结构，不能直接套标准 DES 工具。本题最致命的设计是只有两轮且复用同一子密钥，使每轮的 $F$ 输入输出都能由明密文确定；逆 P 置换后，48 位搜索又自然分解为 8 次 6 位枚举，整体工作量很小。
