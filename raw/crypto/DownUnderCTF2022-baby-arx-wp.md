# DownUnderCTF 2022 baby-arx Writeup

## 题目简述

题目把 64 字节 flag 直接作为流生成器的初始状态。每轮只读取状态最前面的两个字节，分别做异或与移位变换，相加得到一个输出字节，再把输出追加到状态末尾：

```python
b1 = (s0 ^ ((s0 << 1) | (s0 & 1))) & 0xff
b2 = (s1 ^ ((s1 >> 5) | (s1 << 3))) & 0xff
out = (b1 + b2) % 256
state = state[1:] + [out]
```

服务公开前 64 个输出字节。由于前 63 轮尚未轮转到新生成的数据，每个输出只依赖相邻的两个原始 flag 字节。

## 解题过程

已知 flag 以 `D` 开头。若已经恢复当前字节 $s_i$，从输出 $o_i$ 中减去它经过第一种变换后的值，就得到下一个字节经过第二种变换后的结果：

$$
T_2(s_{i+1})\equiv o_i-T_1(s_i)\pmod{256}.
$$

预先枚举可打印字符，建立 $T_2(x)\mapsto x$ 的逆表，然后沿 63 个输出逐字节推进：

```python
from string import printable

inv = {}
for ch in printable:
    x = ord(ch)
    y = (x ^ ((x >> 5) | (x << 3))) & 0xff
    inv[y] = x

flag = b"D"
for out in stream[:-1]:
    x = flag[-1]
    t1 = (x ^ ((x << 1) | (x & 1))) & 0xff
    flag += bytes([inv[(out - t1) % 256]])
```

恢复结果为：

```text
DUCTF{i_d0nt_th1nk_th4ts_h0w_1t_w0rks_actu4lly_92f45fb961ecf420}
```

## 方法总结

题目虽然使用 add、rotate/shift 和 xor，但没有形成密码学意义上的多轮扩散。看到“秘密直接作为移位寄存器状态、每轮仅依赖相邻少数字节”时，应把输出方程写成局部递推；借助已知前缀和受限字符集，通常可以逐字节反演，而不必恢复整个状态或进行指数级搜索。
