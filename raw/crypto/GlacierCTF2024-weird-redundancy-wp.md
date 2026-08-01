# GlacierCTF 2024 WeirdRedundancy

## 题目简述

题目给出三个 `replica_*_challenge.bin` 文件和生成副本的 Go 程序。程序声称使用类似 Shamir Secret Sharing 的多项式冗余，但实际实现把原文件的每个字节扩展成一个 8 字节小端整数，并分别在 $x=1,2,3$ 处求值。

决定性问题不在 ELF 逆向，而在 `evaluate_polynomial()` 的下标使用错误：两个随机系数都只乘了一次 $x$，第二个系数自身被平方，整个表达式对 $x$ 仍然只是一阶函数。因此任意两个副本就能精确恢复原字节。

## 解题过程

### 1. 写出生成器实际计算的公式

核心代码为：

```go
func evaluate_polynomial(secret uint8, a []uint8, x uint32) (y uint64) {
    y = uint64(secret)
    for i, ai := range a {
        y += uint64(x) * uint64(math.Pow(float64(ai), float64(i+1)))
    }
    return
}
```

`k=2`，设两个随机字节为 $a_0,a_1$，则每个原字节 $s$ 的副本值并不是预期的

$$
f(x)=s+a_0x+a_1x^2,
$$

而是

$$
y_x=s+x\left(a_0+a_1^2\right).
$$

令 $m=a_0+a_1^2$，前三个副本就是：

$$
y_1=s+m,\qquad y_2=s+2m,\qquad y_3=s+3m.
$$

所以只需前两个副本：

$$
s=2y_1-y_2.
$$

### 2. 按 8 字节小端整数恢复文件

生成器使用 `binary.LittleEndian.PutUint64()` 写出每个 share，因此每 8 字节对应原文件的一个字节。精确恢复脚本如下：

```python
import struct
from pathlib import Path

r1 = Path("replica_1_challenge.bin").read_bytes()
r2 = Path("replica_2_challenge.bin").read_bytes()
assert len(r1) == len(r2) and len(r1) % 8 == 0

out = bytearray()
for off in range(0, len(r1), 8):
    y1 = struct.unpack_from("<Q", r1, off)[0]
    y2 = struct.unpack_from("<Q", r2, off)[0]
    secret = 2 * y1 - y2
    assert 0 <= secret <= 0xff
    out.append(secret)

Path("reconstructed.bin").write_bytes(out)
```

本地对仓库中的副本执行后，恢复文件与官方 `challenge.bin` 逐字节相同，SHA-256 为：

```text
1c402862d1214c97371ac0cf6617c2e652afbec26e5ebd1517cc736b06464d77
```

### 3. 运行或直接检查恢复的 ELF

恢复结果是正常 ELF，程序将编译时包含的字符串打印出来：

```text
gctf{7h15_f1l3_w45_5h4m1r’s_5ecr375h4r3d}
```

其中 `Shamir’s` 使用的是 Unicode 弯引号 `’`，不是 ASCII 单引号。

仓库中的官方 `reconstrct.py` 用三个点和浮点数做拉格朗日插值。数学上它也在求 $x=0$ 的截距，但大整数经过二进制浮点运算会发生舍入；仓库随附的旧 `reconstructed.bin` 因此有 204 个字节与原文件不同。这里使用整数恒等式 $s=2y_1-y_2$，可避免该实现陷阱。

## 方法总结

本题的关键是依据源码写出“实际公式”，而不是相信 Shamir Secret Sharing 的题面暗示。错误实现把指数加在随机系数上，而没有加在 $x$ 上，使二次多项式退化为直线；两个 share 就足以恢复截距。处理二进制 share 时还应保持全程整数运算，避免用浮点拉格朗日插值破坏本来可精确表示的 64 位数据。
