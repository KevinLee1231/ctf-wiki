# DownUnderCTF 2021 - GammaSafe

## 题目简述

服务要求以十六进制提交一个“签名后的源 IP”，并把解码结果与四字节秘密逐字节比较。每条连接最多尝试三次，但可以重新连接。题目提供的 `GS_strcmp(3)` 手册强调逐字符“校验和纠错”会明显变慢，实际上是在提示比较过程存在时间侧信道。

## 解题过程

服务端的核心比较逻辑如下：

```python
for a, b in zip(SECRET, guess):
    if a != b:
        break
    time.sleep(1.2)
```

若猜测的第一个字节错误，循环立即退出；若前 $k$ 个字节正确，则至少额外等待 $1.2k$ 秒。因此，响应时间泄露了正确前缀长度。`zip` 会在较短输入末尾停止，所以恢复某一位时只需提交“已知前缀 + 一个候选字节”，不必猜完整四字节值。

逐位枚举 `0x00..0xff`，记录从发送到收到 `Incorrect handshake` 的时间。与其他候选相比多出约 1.2 秒的那个字节就是当前位；每三次尝试后重新连接即可绕过单连接限制。为降低网络抖动影响，可以对可疑候选重复测量并使用中位数，而不是依赖一次采样。

```python
from statistics import median
from time import monotonic

def measure(prefix, candidate):
    samples = []
    for _ in range(3):
        io = connect()
        io.recvuntil(b"(hex): ")
        start = monotonic()
        io.sendline((prefix + bytes([candidate])).hex().encode())
        io.recvline()
        samples.append(monotonic() - start)
        io.close()
    return median(samples)

known = b""
for _ in range(4):
    timings = [(measure(known, value), value) for value in range(256)]
    known += bytes([max(timings)[1]])
```

恢复出的秘密为：

```text
26 c6 02 f2
```

提交完整值 `26c602f2` 后得到 flag；其中 `\n` 是 flag 内的两个字面字符，而不是换行：

```text
DUCTF{>[current year]\n>t1m1ng 4774ck5}
```

## 方法总结

逐字节比较在首个不匹配处提前返回，会把正确前缀长度编码进响应时间。本题还人为加入了每字节 1.2 秒延迟，使信号远大于普通网络噪声。通用修复是使用常量时间比较，并让成功、失败路径在长度检查和处理时间上都不泄露前缀信息。
