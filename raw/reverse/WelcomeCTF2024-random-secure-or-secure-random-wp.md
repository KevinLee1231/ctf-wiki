# random secure or secure random?

## 题目简述

程序以当前 Unix 时间调用 `srand(time(NULL))`，随后按 4 字节将 flag 与 libc `rand()` 输出异或并打印十六进制密文。目标是同步时间种子，重放同一伪随机序列并异或还原。

## 解题过程

`rand()` 是确定性 PRNG：相同 libc 实现与相同种子会产生相同序列。服务在播种后还固定 `sleep` 三秒，若在本地启动程序前用同一秒的时间先播种自己的 libc，双方序列即可同步：

```python
import ctypes

libc = ctypes.CDLL("libc.so.6")
libc.srand(libc.time(None))

# 随后立即启动题目程序并取得密文 enc
plain = bytearray()
for off in range(0, len(enc), 4):
    key = libc.rand().to_bytes(4, "little")
    block = enc[off:off + 4]
    plain.extend(a ^ b for a, b in zip(block, key))
```

必须使用与目标相同的 libc `rand()`，因为不同平台的 PRNG 实现未必一致。若无法精确同步秒数，可枚举接收时间附近的小范围时间戳，并用 `grey{` 前缀验证候选。结果为：

```text
grey{r4nd0m_n0t_s0_r4nd0m_owo}
```

## 方法总结

以秒级时间为种子的通用 PRNG 不适合生成密钥流。XOR 本身是可逆的，只要重现伪随机序列就能完全解密；安全用途应改用操作系统 CSPRNG，并避免可猜测种子。
