# Kunmusic

## 题目简述

附件是一个 .NET 启动器。主程序读取资源中的 `data`，把每个字节与 `104` 异或，再通过 `Assembly.Load` 动态加载解密结果。加载出的 DLL 中有一组整数约束；满足约束后，程序用 13 字节循环密钥异或一个数组并显示 flag。

## 解题过程

先还原被包装的程序集：

```python
from pathlib import Path

encrypted = Path("data").read_bytes()
Path("data-decrypted.dll").write_bytes(bytes(value ^ 104 for value in encrypted))
```

输出以 `MZ` 开头，是普通 .NET DLL。用 dnSpy 或 ILSpy 打开后，定位到 `music` 函数：只有一个包含 13 个未知数的巨大条件成立时，程序才会解密数组并弹出消息框。把完整条件转写为 Z3：

```python
from z3 import BitVec, Solver, sat

num = [BitVec(f"num[{index}]", 32) for index in range(13)]
solver = Solver()

solver.add(num[0] + 52296 + num[1] - 26211 + num[2] - 11754 + (num[3] ^ 41236) + num[4] * 63747 + num[5] - 52714 + num[6] - 10512 + num[7] * 12972 + num[8] + 45505 + num[9] - 21713 + num[10] - 59122 + num[11] - 12840 + (num[12] ^ 21087) == 12702282)
solver.add(num[0] - 25228 + (num[1] ^ 20699) + (num[2] ^ 8158) + num[3] - 65307 + num[4] * 30701 + num[5] * 47555 + num[6] - 2557 + (num[7] ^ 49055) + num[8] - 7992 + (num[9] ^ 57465) + (num[10] ^ 57426) + num[11] + 13299 + num[12] - 50966 == 9946829)
solver.add(num[0] - 64801 + num[1] - 60698 + num[2] - 40853 + num[3] - 54907 + num[4] + 29882 + (num[5] ^ 13574) + (num[6] ^ 21310) + num[7] + 47366 + num[8] + 41784 + (num[9] ^ 53690) + num[10] * 58436 + num[11] * 15590 + num[12] + 58225 == 2372055)
solver.add(num[0] + 61538 + num[1] - 17121 + num[2] - 58124 + num[3] + 8186 + num[4] + 21253 + num[5] - 38524 + num[6] - 48323 + num[7] - 20556 + num[8] * 56056 + num[9] + 18568 + num[10] + 12995 + (num[11] ^ 39260) + num[12] + 25329 == 6732474)
solver.add(num[0] - 42567 + num[1] - 17743 + num[2] * 47827 + num[3] - 10246 + (num[4] ^ 16284) + num[5] + 39390 + num[6] * 11803 + num[7] * 60332 + (num[8] ^ 18491) + (num[9] ^ 4795) + num[10] - 25636 + num[11] - 16780 + num[12] - 62345 == 14020739)
solver.add(num[0] - 10968 + num[1] - 31780 + (num[2] ^ 31857) + num[3] - 61983 + num[4] * 31048 + num[5] * 20189 + num[6] + 12337 + num[7] * 25945 + (num[8] ^ 7064) + num[9] - 25369 + num[10] - 54893 + num[11] * 59949 + (num[12] ^ 12441) == 14434062)
solver.add(num[0] + 16689 + num[1] - 10279 + num[2] - 32918 + num[3] - 57155 + num[4] * 26571 + num[5] * 15086 + (num[6] ^ 22986) + (num[7] ^ 23349) + (num[8] ^ 16381) + (num[9] ^ 23173) + num[10] - 40224 + num[11] + 31751 + num[12] * 8421 == 7433598)
solver.add(num[0] + 28740 + num[1] - 64696 + num[2] + 60470 + num[3] - 14752 + (num[4] ^ 1287) + (num[5] ^ 35272) + num[6] + 49467 + num[7] - 33788 + num[8] + 20606 + (num[9] ^ 44874) + num[10] * 19764 + num[11] + 48342 + num[12] * 56511 == 7989404)
solver.add((num[0] ^ 28978) + num[1] + 23120 + num[2] + 22802 + num[3] * 31533 + (num[4] ^ 39287) + num[5] - 48576 + (num[6] ^ 28542) + num[7] - 43265 + num[8] + 22365 + num[9] + 61108 + num[10] * 2823 + num[11] - 30343 + num[12] + 14780 == 3504803)
solver.add(num[0] * 22466 + (num[1] ^ 55999) + num[2] - 53658 + (num[3] ^ 47160) + (num[4] ^ 12511) + num[5] * 59807 + num[6] + 46242 + num[7] + 3052 + (num[8] ^ 25279) + num[9] + 30202 + num[10] * 22698 + num[11] + 33480 + (num[12] ^ 16757) == 11003580)
solver.add(num[0] * 57492 + (num[1] ^ 13421) + num[2] - 13941 + (num[3] ^ 48092) + num[4] * 38310 + num[5] + 9884 + num[6] - 45500 + num[7] - 19233 + num[8] + 58274 + num[9] + 36175 + (num[10] ^ 18568) + num[11] * 49694 + (num[12] ^ 9473) == 25546210)
solver.add(num[0] - 23355 + num[1] * 50164 + (num[2] ^ 34618) + num[3] + 52703 + num[4] + 36245 + num[5] * 46648 + (num[6] ^ 4858) + (num[7] ^ 41846) + num[8] * 27122 + (num[9] ^ 42058) + num[10] * 15676 + num[11] - 31863 + num[12] + 62510 == 11333836)
solver.add(num[0] * 30523 + (num[1] ^ 7990) + num[2] + 39058 + num[3] * 57549 + (num[4] ^ 53440) + num[5] * 4275 + num[6] - 48863 + (num[7] ^ 55436) + (num[8] ^ 2624) + (num[9] ^ 13652) + num[10] + 62231 + num[11] + 19456 + num[12] - 13195 == 13863722)

# 已知明文以 hgame{ 开头，可加入前六个循环密钥字节以加速求解。
solver.add(
    num[0] == 236,
    num[1] == 72,
    num[2] == 213,
    num[3] == 106,
    num[4] == 189,
    num[5] == 86,
)

assert solver.check() == sat
model = solver.model()
key = [model[value].as_long() for value in num]
print(key)
```

模型为：

```text
[236, 72, 213, 106, 189, 86, 62, 53, 120, 199, 15, 93, 133]
```

把 DLL 中的密文数组与该 13 字节序列循环异或：

```python
ciphertext = [
    132, 47, 180, 7, 216, 45, 68, 6, 39, 246,
    124, 2, 243, 137, 58, 172, 53, 200, 99, 91,
    83, 13, 171, 80, 108, 235, 179, 58, 176, 28,
    216, 36, 11, 80, 39, 162, 97, 58, 236, 130,
    123, 176, 24, 212, 56, 89, 72,
]

flag = bytes(value ^ key[index % len(key)] for index, value in enumerate(ciphertext))
print(flag.decode())
```

得到：

```text
hgame{z3_1s_very_u5eful_1n_rever5e_engin3ering}
```

官方 PDF 没有直接写出最终字符串；该结果同时由本地异或复算和 [oacia 的 HGAME 2023 Reverse 复盘](https://oacia.dev/hgame2023-reverse-writeup/) 交叉核对，正文已包含完整还原过程。

## 方法总结

.NET 动态加载题应先追踪 `Assembly.Load` 的字节来源，而不是停留在启动器界面。对大量混合加法、乘法和异或的定长整数约束，Z3 比手工逆推可靠；已知 flag 前缀还可以直接转化为部分密钥约束，显著减少求解空间。
