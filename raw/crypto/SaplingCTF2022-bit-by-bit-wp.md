# Bit by Bit

## 题目简述

服务逐位检查 flag。对于值为 0 的比特，它会调用参数为 $N=2^{16}$ 的 scrypt；对于值为 1 的比特，只计算一次 SHA-256。两条分支返回给客户端的文本相同，但耗时相差很大，因此可以通过响应时间逐位恢复 flag。这是一道典型的时间侧信道题，攻击目标不是逆向哈希，而是识别程序走了哪条分支。

## 解题过程

先对同一位置重复请求若干次，分别观察快、慢两簇耗时。网络抖动会让单次测量失真，因此应使用中位数或截尾均值，而不是看到一次慢响应就判 0。服务端核心逻辑可以概括为：

~~~python
if bit == 0:
    hashlib.scrypt(data, salt=salt, n=2**16, r=8, p=1)
else:
    hashlib.sha256(data).digest()
~~~

逐字节从高位到低位测试，并对每一位收集多次样本：

~~~python
from statistics import median
from time import perf_counter

def measure(index, bit_index, rounds=5):
    samples = []
    for _ in range(rounds):
        start = perf_counter()
        query(index, bit_index)
        samples.append(perf_counter() - start)
    return median(samples)

bits = []
for byte_index in range(flag_length):
    value = 0
    for bit_index in range(7, -1, -1):
        elapsed = measure(byte_index, bit_index)
        bit = int(elapsed < threshold)
        value |= bit << bit_index
    bits.append(value)
print(bytes(bits))
~~~

阈值应由当前连接的基线样本确定；更稳妥的做法是对每个位置追加测量，直到快慢两类的置信区间分离。恢复结果为：

~~~text
maple{s!d3_ch@nn3l_a77@ck5}
~~~

## 方法总结

密码比较、权限判断和密钥处理只要在秘密相关分支中执行了明显不同的工作量，就可能泄露信息。利用时间侧信道时，真正困难的是控制噪声：重复采样、使用稳健统计量、动态校准阈值通常比单纯提高请求频率更有效。防御上应使用常数时间逻辑，并避免在秘密位上选择代价悬殊的算法。
