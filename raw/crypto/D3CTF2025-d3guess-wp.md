# d3guess

## 题目简述

本题是带噪声的猜数反馈接口。服务会根据玩家提交的数给出“大了/小了/赢了”等反馈，但反馈存在噪声，因此普通二分不能把一半区间直接排除。题目分为 `challenge1` 和 `challenge2`：前者用于快速收集随机数输出信息，后者要求在大量轮次中保持高成功率，单靠贝叶斯二分仍不够，需要结合随机数预测。

## 解题过程

预期解是使用贝叶斯算法进行二分：每次收到容器反馈后，不把某一半区间完全否定，而是根据噪声概率给候选值重新分配权重，再选择累计概率接近中位数的位置作为下一次猜测。对于 `challenge1`，找到两个相邻值使容器返回不同即可直接恢复本轮随机数；对于 `challenge2`，使用贝叶斯二分收集足够多的随机数输出，再恢复 MT19937 状态，后续直接预测随机数。由于题目要求在 2200 组中成功 2112 组，单纯概率二分在时间和成功率上都不够，因此需要先用 `challenge1` 的 350 组和部分 `challenge2` 的结果恢复 PRNG 状态。大约需要 40 分钟的服务端交互时间，本地计算约 10 秒。

Exp:

```python
from pwn import *
import math
from tqdm import trange
from gf2bv import LinearSystem
from gf2bv.crypto.mt import MT19937
N = 2**32
wins1 = 0
wins2 = 0
rounds1 = 350
times1 = 32
r = 0.1
rounds2 = 2200
times2 = 64
t = N // 6
u = {0.075: 2**32 - 1, 0.15: 5 * t + 3, 0.225: 4 * t + 2, 0.3: 3 * t + 1, 0.375: 2 * t + 1, 0.45: t}
bs = 32
lin = LinearSystem([32] * 624)
mt = lin.gens()
rng = MT19937(mt)
zeros = []
forecast = False
forecast_list = []
p = remote("34.150.83.54",31570)
# p = process(['python', 'd3guess/chal.py'])
[print(p.recvline()) for _ in '____']
def guess1(times1):
    global wins1
    global forecast_list
    tt = 0
    def s(L):
        nonlocal tt
        L = str(L).encode()
        p.sendlineafter(b'[d3ctf@oracle] give me a number > ', L)
        ss = p.recvline()
        if ss.strip() != b'you win':
            tt += 1
            h = float(ss.decode().split(': ')[-1])
            return h
        else: return 'win'
    L1 = 2**32 - 1
    h1 = s(L1)
    if h1 == 0.45:
        L1 = t
        L2 = 0
        h2 = 0.075
        h1 = 0.15
    else:
        L2 = L1 - t
        if h1 == 0.15:
            h2 = 0.225
        else:
            h2 = h1 + 0.075
    for i in range(times1):
        M = (L1 + L2) // 2
        h = s(M)
        if h == 'win':
            zeros.append(rng.getrandbits(bs) ^ int(M - 1))
            [rng.getrandbits(64) for _ in range(tt)]
            forecast_list.append(tt)
            win1 += 1
            # print('GG', i)
            return
        if h == h1:
            L1 = M
        elif h == h2:
            L2 = M
        else:
            print('error', h1, h2, h)
            exit()
        if abs(L1 - L2) == 1:
            assert h1 != h2
            if L1 < L2:
                L1, L2 = L2, L1
                u[h1], u[h2] = u[h2], u[h1]
            if h1 > h2 and s(L1 + u[h1]) == 'win':
                zeros.append(rng.getrandbits(bs) ^ int(L1 + u[h1] - 1))
                [rng.getrandbits(64) for _ in range(tt)]
                forecast_list.append(tt)
                wins1 += 1
                # print('GG', i)
                return
            elif s(L2 - u[h2]) == 'win':
                zeros.append(rng.getrandbits(bs) ^ int(L2 - u[h2] - 1))
                [rng.getrandbits(64) for _ in range(tt)]
                forecast_list.append(tt)
                wins1 += 1
                # print('GG', i)
                return
    else:
        print('what')
        exit()
def guess2(times2, forecast):
    global wins2
    global forecast_list
    low = 1
    high = (1 << 32) - 1
    segments = [(low, high, 1.0)]
    for attempt in range(times2):
        total_weight = sum(seg[2] for seg in segments)
        if total_weight == 0:
            break
        for i, seg in enumerate(segments):
            a, b, w = seg
            segments[i] = (a, b, w / total_weight)
        cum_prob = 0.0
        guess_val = None
        for (a, b, w) in segments:
            seg_prob = w
            seg_length = b - a + 1
            if cum_prob + seg_prob < 0.5:
                cum_prob += seg_prob
                continue
            frac = (0.5 - cum_prob) / seg_prob
            if frac < 0:
                frac = 0
            if frac > 1:
                frac = 1
            offset = math.floor(frac * seg_length)
            guess_val = a + offset
            break
        if guess_val is None:
            guess_val = segments[-1][1]
        p.sendlineafter(b'[d3ctf@oracle] give me a number > ', str(guess_val).encode())
        resp = p.recvline().decode().strip()
        if resp == 'you win':
            wins2 += 1
            zeros.append(rng.getrandbits(bs) ^ int(guess_val - 1))
            [rng.getrandbits(64) for _ in range(attempt)]
            forecast_list.append(attempt)
            if forecast:
                [pyrand.random() for _ in range(attempt)]
            break
        elif resp == 'your number is too small':
            feedback = "higher"
        elif resp == 'your number is too big':
            feedback = "lower"
        new_segments = []
        for (a, b, w) in segments:
            if guess_val < a or guess_val > b:
                if feedback == "higher":
                    if b < guess_val:
                        new_w = w * r
                    else:
                        new_w = w * (1 - r)
                else:
                    if b < guess_val:
                        new_w = w * (1 - r)
                    else:
                        new_w = w * r
                new_segments.append((a, b, new_w))
            else:
                left_a = a
                left_b = guess_val - 1
                right_a = guess_val + 1
                right_b = b
                left_len = max(0, guess_val - a)
                right_len = max(0, b - guess_val)
                if feedback == "higher":
                    if left_len > 0:
                        left_w = w * (left_len / (b - a + 1)) * r
                        new_segments.append((left_a, left_b, left_w))
                    if right_len > 0:
                        right_w = w * (right_len / (b - a + 1)) * (1 - r)
                        new_segments.append((right_a, right_b, right_w))
                else:
                    if left_len > 0:
                        left_w = w * (left_len / (b - a + 1)) * (1 - r)
                        new_segments.append((left_a, left_b, left_w))
                    if right_len > 0:
                        right_w = w * (right_len / (b - a + 1)) * r
                        new_segments.append((right_a, right_b, right_w))
        segments = new_segments
    else:
        # print(f"GG")
        rng.getrandbits(32)
        [rng.getrandbits(64) for _ in range(times2)]
        forecast_list.append(times2)
        if forecast:
            [pyrand.random() for _ in range(times2)]
for _ in trange(rounds1):
    guess1(times1)
print(f"Total Wins: {wins1} / {rounds1} rounds")
for round_idx in trange(rounds2):
    # guess(times)
    if len(zeros) >= 624 and not forecast:
        zeros.append(mt[0] ^ int(0x80000000))
        sol = lin.solve_one(zeros)
        if sol:
            print(len(zeros))
            assert len(forecast_list) == round_idx + rounds1
            rng = MT19937(sol)
            pyrand = rng.to_python_random()
            [[pyrand.randint(1, N - 1)] + [pyrand.random() for _ in range(forecast_list[i])] for i in range(round_idx + rounds1)]
            forecast = True
    if forecast:
        guess_val = pyrand.randint(1, N - 1)
        p.sendlineafter(b'[d3ctf@oracle] give me a number > ', str(guess_val).encode())
        resp = p.recvline().decode().strip()
        if resp == 'you win':
            # print(f"forecast {guess_val} success")
            wins2 += 1
        else:
            # print(f'forecast {guess_val} fail')
            guess2(times2 - 1, forecast)
    else:
        guess2(times2, forecast)
print(f"Total Wins: {wins2} / {rounds2} rounds")
p.interactive()
```

## 方法总结

- 核心技巧：噪声二分不能做硬切分，要用贝叶斯权重维护候选空间；成功轮次产生的随机数关系可用于恢复 MT19937 状态。
- 识别信号：如果猜数反馈存在固定噪声率，并且轮次很多、成功阈值很高，应同时考虑概率搜索和 PRNG 状态恢复。
- 复用要点：先用低成本阶段收集足够多的可恢复输出，再把恢复出的随机数状态用于后续直接预测；否则仅靠贝叶斯搜索很容易被轮次成功率卡死。
