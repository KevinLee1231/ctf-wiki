# Guardians of the Kernel

## 题目简述

题目给出 Linux 内核镜像 `bzImage` 和压缩的 initramfs。启动后需要与 `/proc/Flag-Checker` 交互，依次解开三层检查。真正的校验逻辑不在用户态程序中，而在 initramfs 内的 `flag_checker.ko` 内核模块里。

三层分别校验固定前缀、七位十进制数字和经过逐字节变换的结尾。官方 `sol.c` 采用穷举中间七位的方式；下文使用位向量约束直接恢复它，并对照[参赛者的 Z3 分析](https://mcfx.us/posts/2023-09-01-sekaictf-2023-writeup/#guardians-of-the-kernel)核验运算宽度。

## 解题过程

先从 initramfs 中取出模块。该压缩包可被 GNU tar 识别为 cpio，因此无需启动虚拟机：

```bash
tar -tf initramfs.cpio.gz | grep flag_checker
tar -xf initramfs.cpio.gz ./flag_checker.ko
```

将 `flag_checker.ko` 载入 IDA，字符串交叉引用会把分析引向 `device_ioctl`。模块注册的 procfs 节点是 `/proc/Flag-Checker`，三个有效 ioctl 编号依次为 `0x7000`、`0x7001` 和 `0x7002`；后两层还会检查前一层是否已经成功。

第一层从用户空间复制 6 字节，并分别与 `0x414b4553` 和 `0x7b49` 比较。按小端序还原就是：

```text
SEKAI{
```

第二层复制 7 字节，先要求每个字节都在 ASCII `0` 至 `9` 之间，再对两个 32 位分组进行乘法、循环移位和异或混合。这里必须按 32 位无符号整数处理溢出，不能把反编译表达式直接当作无限精度整数。下面的 Z3 位向量脚本忠实复现检查：

```python
from z3 import BitVec, LShR, RotateLeft, RotateRight, Solver, sat

buffer = [BitVec(f"t{i}", 32) for i in range(7)]
solver = Solver()

for x in buffer:
    solver.add(x >= ord("0"), x <= ord("9"))

v8 = 7 * RotateLeft(
    1507359807
    * RotateRight(
        422871738
        * (
            buffer[0]
            | (buffer[1] << 8)
            | (buffer[2] << 16)
            | (buffer[3] << 24)
        ),
        15,
    ),
    11,
)
v9 = RotateRight(
    422871738
    * ((buffer[5] << 8) ^ (buffer[6] << 16) ^ buffer[4]),
    15,
)
mixed = (v8 + 1204333666) ^ (1507359807 * v9)
v10 = 1984242169 * (mixed ^ 7 ^ LShR(mixed, 16))
final = (2**32 - 1817436554) * (LShR(v10, 13) ^ v10)

solver.add((LShR(final, 16) ^ final) == 261736481)
assert solver.check() == sat

model = solver.model()
print(bytes(model[x].as_long() for x in buffer))
```

输出为：

```text
b'6001337'
```

第三层复制 12 字节，并按索引从前向后执行：

```c
buffer[i] += buffer[i + 1] * ~i;
```

变换后的前 12 字节应等于：

```text
0e af 88 1d b9 88 8c 78 ec 11 f3 7d
```

因为第 `i` 项依赖尚未变换的第 `i+1` 项，解密时要从末尾逆序。补上校验所需的 NUL 哨兵后可直接还原：

```python
encrypted = list(
    (0x788C88B91D88AF0E).to_bytes(8, "little")
    + (0x7DF311EC).to_bytes(4, "little")
    + b"\0"
)

for i in range(11, -1, -1):
    encrypted[i] = (
        encrypted[i] - encrypted[i + 1] * ~i
    ) % 256

print(bytes(encrypted[:-1]).decode())
```

得到结尾 `SEKAIPL@YER}`。依次提交三段即可通过所有层：

```c
ioctl(fd, 0x7000, "SEKAI{");
ioctl(fd, 0x7001, "6001337");
ioctl(fd, 0x7002, "SEKAIPL@YER}");
```

拼接得到：

```text
SEKAI{6001337SEKAIPL@YER}
```

## 方法总结

面对 `bzImage + initramfs`，应先拆 initramfs 并定位自定义模块，通常比从整个内核镜像开始逆向更高效。本题的状态机、procfs 节点和三段校验都集中在一个 ioctl 处理函数中。固定比较量按小端序还原，线性字节变换逆序求解；包含溢出和循环移位的散列则使用固定宽度位向量建模，能把一千万次穷举压缩成一次约束求解。
