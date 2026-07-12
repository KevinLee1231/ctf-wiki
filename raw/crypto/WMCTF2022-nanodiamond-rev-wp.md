# nanoDiamond - rev

## 题目简述

题目服务端 `app.py` 是一个 50 轮的真假箱子游戏。每轮有 6 个布尔变量 `B0` 到 `B5` 表示箱子状态，玩家有 13 次提问机会，提问表达式只能由 `==`、括号、`0/1`、`and/or` 和 `B0` 到 `B5` 组成；服务端会随机把其中 0 到 2 次回答取反，即“Skeleton Merchant can lie twice”。每轮结束必须一次性提交 6 个箱子的真实状态。解题目标是设计带纠错能力的查询，使 13 次回答足以恢复 6 个布尔值，连续通过 50 轮拿 flag。

## 解题过程

服务端允许问单个变量真假，也允许问两个变量组成的等式/合取表达式。exp 的策略是先问 6 个单变量得到初始猜测，再问三组“两两是否同时等于当前猜测”的校验问题。根据三组校验中失败的数量判断谎言分布：

- 三组都通过时，说明当前猜测大概率正确，只再消耗若干问题对前几个变量做冗余确认。
- 只有一组失败时，说明那一组内可能有变量被谎言污染，直接重问该组两个变量。
- 两组或三组异常时，对异常组变量做多数投票。

由于每轮最多只有 2 次谎言，单变量查询和成对校验结合后可把错误定位到较小范围，再用剩余问题纠正。

exp如下：

```python
from rich.progress import track
from pwn import *
import string
import hashlib

context(log_level='info', os='linux', arch='amd64')

r = remote('127.0.0.1', 65100)


def guess_remote(expr):
    global r
    r.recvuntil(b'Question: ')
    r.sendline(expr)
    r.recvuntil(b'Answer: ')
    ret = r.recvuntil(b'!', drop=True)
    return int(ret == b'True')

if __name__ == '__main__':

    def proof_of_work_2(suffix, hash):  # sha256, suffix, known_hash
        def judge(x): return hashlib.sha256(
            x.encode()+suffix).digest().hex() == hash
        return util.iters.bruteforce(judge, string.digits + string.ascii_letters, length=4)
    r.recvuntil(b'sha256(XXXX+')
    suffix = r.recv(16)
    r.recvuntil(b' == ')
    hash = r.recv(64).decode()
    r.recvuntil(b'Give me XXXX:')
    r.sendline(proof_of_work_2(suffix, hash))

    for round in track(range(50)):
        chests = [guess_remote(expr) for expr in ['B0', 'B1', 'B2', 'B3', 'B4', 'B5']]
        pairs = tuple([guess_remote(expr) for expr in [f'B0 == {chests[0]} and B1 == {chests[1]}', f'B2 == {chests[2]} and B3 == {chests[3]}', f'B4 == {chests[4]} and B5 == {chests[5]}']])
        if sum(pairs) == 3:
            print("CASE 0")
            tuple([guess_remote(expr) for expr in ['B0', 'B1', 'B2', 'B3']])
        elif sum(pairs) == 2:
            print("CASE 1")
            if pairs[0] == 0:
                chests[0] = int(chests[0] + guess_remote("B0") + guess_remote("B0") > 1)
                chests[1] = int(chests[1] + guess_remote("B1") + guess_remote("B1") > 1)
            if pairs[1] == 0:
                chests[2] = int(chests[2] + guess_remote("B2") + guess_remote("B2") > 1)
                chests[3] = int(chests[3] + guess_remote("B3") + guess_remote("B3") > 1)
            if pairs[2] == 0:
                chests[4] = int(chests[4] + guess_remote("B4") + guess_remote("B4") > 1)
                chests[5] = int(chests[5] + guess_remote("B5") + guess_remote("B5") > 1)
        elif sum(pairs) == 1:
            print("CASE 2")
            if pairs[0] == 0:
                chests[0], chests[1] = guess_remote("B0"), guess_remote("B1")
            if pairs[1] == 0:
                chests[2], chests[3] = guess_remote("B2"), guess_remote("B3")
            if pairs[2] == 0:
                chests[4], chests[5] = guess_remote("B4"), guess_remote("B5")
        print(chests)
        r.recvuntil('open the chests:')
        r.sendline(' '.join([str(int(x)) for x in chests]))

    r.interactive()
```

## 方法总结

本题核心是一个带最多 2 次谎言的交互式布尔查询纠错问题。服务端 `eval` 表达式白名单虽然限制了语法，但仍允许构造单点查询和成对一致性校验。

识别信号是固定 6 个布尔变量、13 次回答、最多 2 次谎言、连续 50 轮且每轮独立。查询数量明显多于 6，说明题目希望利用冗余校验而不是暴力猜。

复现时先问 `B0` 到 `B5` 得初值，再问三组配对校验定位异常组；对异常组重问或做多数投票。PoW 通过后自动循环 50 轮即可进入交互并打印 flag。
