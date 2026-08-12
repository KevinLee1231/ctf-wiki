# DownUnderCTF 2022 last-digit Writeup

## 题目简述

服务把 flag 字节按大端解释成整数 $F<2^{1024}$。用户输入整数 $x$ 后，程序计算 $F\cdot x$，转成十进制字符串并只返回最后一位。直接观察个位最多只能得到模 10 信息；真正可用的是 Python 3.10.7 对超长十进制整数转换的 4300 位限制，以及异常处理泄露出的单调 oracle。

本题属于纯运行时数值谜题，不能稳定映射到 14 个安全技术方向，因此暂存 `_unclassified`。

## 解题过程

服务代码把输入解析和十进制转换放在同一个 `try` 中：

```python
try:
    n = FLAG * int(input('> '))
    print('Your digit is:', str(n)[-1])
except ValueError:
    print('Not a valid number! >:(')
```

Python 为缓解超大整数与十进制字符串转换的拒绝服务问题，默认拒绝超过 4300 位的转换。因此即使输入本身是合法整数，只要乘积太大，`str(n)` 也会抛出 `ValueError`。令
$U=10^{4300}$，则响应近似提供谓词：

```text
正常返回  <=>  F * x < U
ValueError <=>  F * x >= U
```

临界乘数是 $x^*=\lceil U/F\rceil$。由 $F<2^{1024}$ 可知
$U/2^{1024}<x^*\le U$，可以在这个区间对异常响应做二分：

```python
from pwn import remote

io = remote('target', 1337)

def overflow(x):
    io.sendlineafter(b'> ', str(x).encode())
    return b'>:(' in io.recvline()

U = 10**4300
low = U // 2**1024
high = U

for _ in range(1024):
    mid = (low + high) // 2
    if overflow(mid):
        high = mid - 1
    else:
        low = mid + 1

threshold = mid
flag_int = U // threshold + 1
flag = flag_int.to_bytes(128, 'big').lstrip(b'\0')
print(flag.decode())
```

1024 轮并不是要写出 $x^*$ 的全部约 14000 个二进制位，而是把相对误差压到足以唯一确定一个小于 $2^{1024}$ 的 $F$。恢复结果为：

```text
CTF{14288_bits_should_be_enough_for_anybody_:)}
```

该题官方 flag 前缀确实是 `CTF{`，不是 `DUCTF{`。

## 方法总结

可见的“最后一位”是诱饵，异常分支才是高带宽信息源。把 Python 的十进制位数上限转成乘积是否越过 $10^{4300}$ 的比较 oracle 后，问题等价于估计 $U/F$，再取倒数恢复 $F$。分析此类服务时应检查输出转换、格式化和异常路径，而不只研究明确打印的数值。
