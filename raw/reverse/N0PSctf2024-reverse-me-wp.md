# Reverse Me

## 题目简述

附件名为 `img.jpg`，但图片查看器无法打开。查看可打印字符串时，可以看到 `tnemmoc...`、`CCG` 等明显反向文本。题目标题和描述也提示“reverse”，因此应把整个文件按字节逆序，而不是尝试修复 JPEG 头。

逆序后得到 x86-64 PIE ELF。程序接收四个整数，只有它们满足一组线性方程时才输出 flag。

## 解题过程

### 逐字节反转附件

```python
from pathlib import Path


data = Path("img.jpg").read_bytes()
Path("rev.elf").write_bytes(data[::-1])
```

检查恢复文件：

```bash
file rev.elf
chmod +x rev.elf
```

结果为 64 位小端 PIE ELF。原附件并不承载可见图片，因此无需保留图片资源。

### 从验证函数建立方程组

反编译后的判断可以整理为：

$$
-10x_1+4x_2+x_3+3x_4=28
$$

$$
-8x_1+9x_2+6x_3-2x_4=72
$$

$$
-2x_1-3x_2-8x_3+x_4=29
$$

$$
5x_1+7x_2+x_3-6x_4=88
$$

使用符号求解：

```python
from sympy import Eq, solve, symbols


x1, x2, x3, x4 = symbols("x1 x2 x3 x4")
equations = [
    Eq(-10 * x1 + 4 * x2 + x3 + 3 * x4, 28),
    Eq(-8 * x1 + 9 * x2 + 6 * x3 - 2 * x4, 72),
    Eq(-2 * x1 - 3 * x2 - 8 * x3 + x4, 29),
    Eq(5 * x1 + 7 * x2 + x3 - 6 * x4, 88),
]

print(solve(equations, (x1, x2, x3, x4)))
```

唯一解为：

$$
(x_1,x_2,x_3,x_4)=(-3,8,-7,-9)
$$

把四个整数按程序要求作为命令行参数：

```bash
./rev.elf -3 8 -7 -9
```

程序输出：

```text
N0PS{r1CKUNr0111N6}
```

## 方法总结

- 核心技巧：先依据反向字符串把整个文件逐字节逆序恢复 ELF，再把验证逻辑转写为线性方程组。
- 识别信号：扩展名与文件内容不符，字符串整体倒序，题面又明确暗示 reverse；恢复后的 ELF 魔数提供强验证。
- 复用要点：文件级逆序应覆盖全部字节，不能只反转每行或字符串。求出参数后必须让恢复的原程序实际走到成功分支，避免抄错系数或参数顺序。
