# SuanHash

## 题目简述

SuanHash 是 64-bit rate/64-bit capacity 的 sponge-like 哈希，但输出结构泄露了 rate 和与上一块相关的 capacity 信息。构造两条只在 rate 部分不同的消息可恢复状态差分，再追加互补块制造碰撞。

## 解题过程

### 关键观察

SuanHash 是 64-bit rate/64-bit capacity 的 sponge-like 哈希，但输出结构泄露了 rate 和与上一块相关的 capacity 信息。

### 求解步骤

SuanHash is sponge-like with a 64-bit rate and 64-bit capacity. During absorption, each
message block state ^ block is permuted by AES. During squeezing, the output high half
always mirrors the current rate bits, while the lower half is (current_cap ^ prev_block_low)—
the stored last_low is simply the pad of the previous block.
Feed two almost identical one-and-a-bit–block messages (A…AB and A…AC). Their padded
final blocks share the same first 15 bytes and differ only in the rate portion, so from two
digests we recover Δ = (hi1 ⊕ hi2)‖((lo1 ⊕ lo2 ⊕ pad1_low ⊕ pad2_low)), which equals the
XOR difference between the two internal states after the final permutation.
Because absorption of the next block is just permute(state ⊕ next_block), appending block X
to the first message and X ⊕ Δ to the second keeps the two permute inputs identical, so the
states—and therefore the digests—collide while the strings remain different.
from __future__ import annotations

import argparse
import os
import sys

from pwn import log, process, remote

BLOCK = 16
MASK64 = (1 << 64) - 1

def pad_bytes(msg: bytes) -> bytes:
    if not msg:
        data = b"\\x80" + b"\\x00" * (BLOCK - 1)
    else:
        data = msg + b"\\x80"
        data += b"\\x00" * (-len(data) % BLOCK)
    return data

def recv_prompt(conn, idx: int):
    prompt = f"MSG {idx} (hex): ".encode()
    conn.recvuntil(prompt)

def read_hash_line(conn) -> str:
    line = conn.recvline()
    while line and not line.startswith(b"H = "):
        line = conn.recvline()
    if not line:
        raise EOFError("connection closed while waiting for hash output")
    return line.split(b"=", 1)[1].strip().decode()

def query(conn, idx: int, msg: bytes) -> int:
    recv_prompt(conn, idx)
    conn.sendline(msg.hex().encode())
    h_hex = read_hash_line(conn)
    return int(h_hex, 16)

def build_collision(conn, round_idx: int):
    prefix = b"A" * BLOCK
    tail1, tail2 = b"B", b"C"
    msg1 = prefix + tail1
    msg2 = prefix + tail2
    pad1 = pad_bytes(msg1)
    pad2 = pad_bytes(msg2)

    h1 = query(conn, 1, msg1)
    h2 = query(conn, 2, msg2)

    u1, u2 = h1 >> 64, h2 >> 64
    d1, d2 = h1 & MASK64, h2 & MASK64
    b1_low = int.from_bytes(pad1[-8:], "big")
    b2_low = int.from_bytes(pad2[-8:], "big")
    low = (d1 ^ d2 ^ b1_low ^ b2_low) & MASK64
    delta = ((u1 ^ u2) << 64) | low

    block_c = os.urandom(BLOCK)
    block_c_prime = (int.from_bytes(block_c, "big") ^ delta) & ((1 << 128) -
1)

    msg3 = pad1 + block_c
    msg4 = pad2 + block_c_prime.to_bytes(BLOCK, "big")

    h3 = query(conn, 3, msg3)
    h4 = query(conn, 4, msg4)

### 跨页补回：碰撞脚本收尾

if h3 != h4:
        raise ValueError(f"collision failed in round {round_idx}")

def solve(conn, rounds: int):
    banner = conn.recvuntil(b"[Round 1]")
    log.info("Connected: %s", banner.decode(errors="ignore").splitlines()[-1])

    for rnd in range(1, rounds + 1):
        build_collision(conn, rnd)
        log.info("Round %d completed", rnd)

    rest = conn.recvall(timeout=2)
    if rest:
        print(rest.decode(errors="ignore"))

def main():
    parser = argparse.ArgumentParser(description="Exploit SuanHash collision
rounds")
    parser.add_argument("--host", default="<challenge-host>")
    parser.add_argument("--port", type=int, default=42103)
    parser.add_argument("--local", action="store_true", help="connect to local
python task")
    parser.add_argument("--rounds", type=int, default=500)
    args = parser.parse_args()

    if args.local:
        conn = process([sys.executable, "task.py"])
    else:
        conn = remote(args.host, args.port)

    with conn:
        solve(conn, args.rounds)

if __name__ == "__main__":
    main()

## 方法总结

- 核心技巧：利用 sponge 输出泄露状态差分构造多轮碰撞。
- 识别信号：digest 的高低半部分分别直接暴露 rate/capacity 派生值。
- 复用要点：比较两条几乎相同消息的 digest，恢复差分后用下一块抵消。
