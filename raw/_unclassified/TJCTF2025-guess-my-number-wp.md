# guess-my-number

## 题目简述

服务在 1 到 1000 的闭区间内随机选择整数，最多允许猜 10 次；每次错误都会明确回答 `Too high` 或 `Too low`。由于 $\lceil\log_2 1000\rceil=10$，反馈恰好足以用二分查找保证命中。

## 解题过程

维护仍可能的闭区间 `[low, high]`，每轮猜中点。若回复过高，则把上界改为 `mid - 1`；若过低，则把下界改为 `mid + 1`。每次都把候选数近似减半，10 轮可区分最多 $2^{10}=1024$ 个值。

```python
from pwn import remote

io = remote("challenge-host", 12345)
low, high = 1, 1000

for _ in range(10):
    middle = (low + high) // 2
    io.recvuntil(b": ")
    io.sendline(str(middle).encode())
    response = io.recvline().decode()

    if "Too high" in response:
        high = middle - 1
    elif "Too low" in response:
        low = middle + 1
    else:
        print(response.strip())
        break
```

对应的 flag 为：

```text
tjctf{g0od_j0b_u_gu33sed_correct_998}
```

## 方法总结

- 核心技巧：利用大小反馈执行闭区间二分查找。
- 识别信号：范围大小不超过 $2^k$，且正好给出 $k$ 次 high/low 比较机会。
- 复用要点：更新边界时必须排除已经猜过的中点，否则最后两个候选可能陷入重复；网络脚本也要先同步提示符再发送。
