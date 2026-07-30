# L3akCTF 2024 Anti Writeup

## 题目简述

题目把一个 16 字节输入分成四个 4 字节大端整数，并用

$$
Q(b,r)=b\oplus\operatorname{ROL}_{32}(b,r)
$$

逐块校验。真正干扰分析的是一套名为 HOP（Hook Oriented Programming）的反调试结构：程序连续 `fork` 两次，两个父进程充当 tracer，最内层子进程才执行校验逻辑。子进程主动调用 `PTRACE_TRACEME`，普通调试器无法再直接附加。

项目附带的 HOP 论文和源码给出了完整机制：定制的 dietlibc 不让 `write` 直接使用系统调用号 1，而是临时生成一个伪系统调用号；父进程在每次系统调用停止时读取寄存器，如果 `orig_rax` 等于该伪编号，就把它改回 1 后继续执行。也就是说，输出功能依赖父进程的动态修复，静态查看子进程会看到一套故意错误的系统调用映射。

## 解题过程

### 还原父子进程与系统调用钩子

`main` 的调用关系是：

```text
main
└── tracer
    └── program_start2
        ├── tracer
        └── tracee
```

`tracee` 先声明自己接受父进程跟踪：

```c
ptrace(PTRACE_TRACEME, 0, 0, 0);
raise(SIGCONT);
```

tracer 则反复以 `PTRACE_SYSCALL` 运行子进程，并在系统调用入口检查 `orig_rax`：

```c
short fake_write = syscall_table_rand[1];

if (regs.orig_rax == (unsigned long long)fake_write) {
    regs.orig_rax = 1;
    ptrace(PTRACE_SETREGS, child_pid, 0, &regs);
}
```

论文中的时序图与源码一致：子进程因系统调用而停止，内核通知父进程，父进程读取并修改寄存器，再让子进程继续。这个设计既占用了 `ptrace` 跟踪关系，又把真实系统调用号藏到了父进程中。

预期解法可以实现自己的 tracer，记录随机系统调用映射和运行时随机数。不过本题把校验源码和固定种子一并给出，因此无需完整复刻调试器。

### 确定旋转位数

校验前执行：

```c
srand(RANDOM_VALUE);  // RANDOM_VALUE == 42
int random_number = rand();
```

项目使用 dietlibc 的 Park-Miller `rand_r`：

```c
X = 48271 * (X % 44488) - 3399 * (X / 44488);
```

对种子 42，第一次结果为 `2027382`。x86 的 32 位可变移位只取计数低 5 位，所以实际旋转量为

$$
r=2027382\bmod 32=22.
$$

还可以直接用四个目标常量和已知 flag 格式验证：只有左旋 22 位能同时生成

```text
518016411, 4152142508, 1156902051, 2196107258
```

官方说明里的示例恢复函数使用步长 17，这与当前仓库的 `RANDOM_VALUE`、dietlibc 实现和四个目标常量并不一致，不能原样照搬。

### 逐块逆变换

以最低位为第 0 位。对于 $q=b\oplus\operatorname{ROL}_{32}(b,22)$，有

$$
q_i=b_i\oplus b_{(i-22)\bmod 32}.
$$

因为 $\gcd(22,32)=2$，位依赖图分成两个长度为 16 的环。分别猜测两个环的起始位后即可传播其余位，所以每个目标最多只有 4 个原像，不需要遍历 $2^{32}$ 个整数。

完整恢复脚本如下：

```python
from itertools import product

targets = [
    518016411,
    4152142508,
    1156902051,
    2196107258,
]
rotation = 22


def invert_q(q):
    answers = []

    # 偶数位和奇数位各形成一个环，分别猜一个起始位。
    for seeds in product((0, 1), repeat=2):
        bits = [None] * 32
        valid = True

        for start, seed in enumerate(seeds):
            bits[start] = seed
            previous = start

            while True:
                current = (previous + rotation) % 32
                value = ((q >> current) & 1) ^ bits[previous]

                if current == start:
                    valid &= value == bits[start]
                    break

                if bits[current] is not None:
                    valid = False
                    break

                bits[current] = value
                previous = current

        if valid:
            number = sum(bit << i for i, bit in enumerate(bits))
            answers.append(number.to_bytes(4, "big"))

    return answers


chunks = []
for target in targets:
    printable = [
        candidate
        for candidate in invert_q(target)
        if all(0x20 <= byte < 0x7f for byte in candidate)
    ]
    print(printable)
    chunks.append(printable)
```

输出为：

```text
[b'L3AK']
[b'{br0', b".7'e"]
[b'_c4n']
[b'_r3v']
```

第二块存在两个可打印原像，但只有 `{br0` 符合已知 flag 前缀。程序只比较前 16 字节，因此拼出的是 `L3AK{br0_c4n_r3v`，补上格式要求的右花括号后得到：

```text
L3AK{br0_c4n_r3v}
```

## 方法总结

- HOP 的核心不是修改程序代码，而是由已占用 `ptrace` 关系的父进程在系统调用边界改写寄存器；分析时应同时检查 tracee 与 tracer。
- 固定种子、具体 `rand` 实现和 CPU 移位语义缺一不可。不能把宿主 glibc 的 `rand()` 结果套到题目自带的 dietlibc 上。
- $b\oplus\operatorname{ROL}(b,r)$ 是 GF(2) 上的线性变换。根据 $\gcd(r,32)$ 划分位环，只猜每个环的一个起始位即可恢复全部候选。
- 官方说明中的示例参数与当前源码不一致时，应以源码、目标常量和正向复算结果为准。
