# SplitMaster

## 题目简述

题目相当于要求解 m=512bit,leak=30bit 的 hnp 问题，但 30bit 并不连续，最大连续段设置为 25，卡了界，不能通过单一段落进行 hnp，也限制了不能放在首尾。可拆分 30bit 为两个 15bit，通过增加切分段数可以扩大格的维度，格子构建为基本 hnp 格。

附件 `task.py` 是交互 oracle：服务端生成 512 bit 的 `key` 和 512 bit 素数 `q`，先把 `q` 发给选手。每轮随机生成素数 `a`，计算 `b=a*key mod q`，然后允许选手提交一组 `segment_bits`，服务端返回 `a` 和 `split_master(b, segment_bits)`。总共只有 20 次查询，最后需要提交完整 `key` 才能拿到 flag。因此解法不是固定输出题，而是通过自选切分方式主动构造多组不连续 bit 泄露。

## 解题过程

交互时选择切分参数 `161 15 161 15 160`，让 512 bit 隐藏数被分成五段，其中第 2、4 段各泄露 15 bit。利用多组样本，选一条作为基准，将其他样本与基准相减消去完整隐藏数中的公共部分，把未知段转成格中的短向量。脚本对不同基准样本枚举，构造维度 `3*n+4` 的格后用 BKZ 规约，命中带非零末坐标的向量时恢复候选 key 并提交。

```python
from Crypto.Util.number import *

from pwn import *
context.log_level='debug'
io=remote("127.0.0.1","10003")
io.recvuntil(b'q:')
q=int(io.recvline().decode(),10)

b2b4=[]
A=[]
for i in range(20):
  io.recvuntil(b'> ')
  payload=b'161 15 161 15 160'
  io.sendline(payload)
  io.recvuntil(b'a:')
  a=int(io.recvline().decode(),10)
  io.recvuntil(b'gift:')
  gift=eval(io.recvline().decode())
  A.append(a)
  b2b4.append(gift)
s1,s2,s3,s4,s5=[161,15,161,15,160]

m = 512
T1 = 2^(s2+s3+s4+s5)
T2 = 2^(s3+s4+s5)
T3 = 2^(s4+s5)
T4 = 2^(s5)

assert len(A) == len(b2b4)
Aorg = [x for x in A]
b2org = [x[0] for x in b2b4]
b4org = [x[1] for x in b2b4]


nn = ceil((s1+s3+s5)/((s2+s4))) + 3
print(nn)
A = [x for x in Aorg[:nn]]
b2 = [x for x in b2org[:nn]]
b4 = [x for x in b4org[:nn]]
n = len(A)-1

AA = [x for x in A]
bb2 = [x for x in b2]
bb4 = [x for x in b4]
for choice in range(n):
  A = [x for x in AA]
  b2 = [x for x in bb2]
  b4= [x for x in bb4]

  A0 = A[choice]
  A0i = inverse_mod(A0,q)
  b02 = b2[choice]
  b04 = b4[choice]
  del A[choice]
  del b2[choice]
  del b4[choice]
  assert gcd(A0, q) == 1

  Mt = matrix(ZZ, 3*n+4)
  for i in range(n):
    Mt[3*i, 3*i]  = -q
    Mt[3*i+1, 3*i]  = -T3
    Mt[3*i+2, 3*i]  = -T1
    Mt[3*i+1, 3*i+1]  = 1
    Mt[3*i+2, 3*i+2]  = 1
    Mt[-4, 3*i] = A0i*A[i] % q
    Mt[-3, 3*i] = T3*A0i*A[i] % q
    Mt[-2, 3*i] = T1*A0i*A[i] % q
    Mt[-1, 3*i] = A0i*(T2*(A[i]*b02 - A0*b2[i]) + T4*(A[i]*b04 - A0*b4[i])) % q
  Mt[-4, -4] = 1
  Mt[-3, -3] = 1
  Mt[-2, -2] = 1
  R = 2^(s1-1)
  Mt[-1, -1] = R
  L = Mt.BKZ(block_size=36)


  for l in L:
    if l[-1]:
      b0 = vector(l)

      b01 = b0[-2]
      b03 = b0[-3]
      b05 = b0[-4]
      x0 = (T1*b01 + T2*b02 + T3*b03 + T4*b04 + b05) * A0i % q

      io.recvuntil(b'the key to the flag is: ')
      io.sendline(str(x0%q).encode())
      print(io.recvline())
      exit(0)
```

## 方法总结

- 核心技巧：不连续 bit 泄露的 HNP，需要主动把泄露段拆分为多个小段，通过样本差分和增维格恢复隐藏数。
- 识别信号：泄露 bit 总量看似足够，但被题目限制为不连续且单段长度卡界时，应考虑多段 HNP 格，而不是强行套单段 MSB/LSB 模板。
- 复用要点：切分参数决定格维数和短向量形态；选基准样本时要枚举，因为不同样本会影响格规约质量。恢复候选后必须马上提交或代回验证。
