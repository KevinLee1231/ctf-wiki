# TSGCTF2021 Beginner's Rev 2021 WP

## 题目简述

题目给出一个 64 位 Linux 程序，要求输入恰好 32 个字符。程序表面上只输出一次 `correct` 或 `wrong`，但真正的校验被拆给一棵由 `fork()` 生成的进程树：连续调用 5 次 `fork()` 后共有 $2^5=32$ 个进程，每个进程只检查输入中的一个位置。

决定性线索不是复杂的字符变换，而是进程树产生的系统调用。子进程虽然把标准输出重定向到 `/dev/null`，`strace -f` 仍然可以观察它们执行的 `write` 系统调用，因此可以把“内部有多少个进程走到 `correct` 分支”变成字符爆破的反馈。

## 解题过程

反编译 `check` 后可以把核心逻辑还原为：

```c
int index = 0, count = 0;
for (int i = 0; i < 5; i++) {
    if (fork()) {
        count++;
    } else {
        count = 0;
        index |= 1 << i;
        dup2(open("/dev/null", O_WRONLY), 1);
    }
}

int result = !is_correct(flag[index], index);
while (count--) {
    int status;
    wait(&status);
    result |= WEXITSTATUS(status);
}
puts(result == 0 ? "correct" : "wrong");
```

每次进入子分支都会把对应位写入 `index`，所以 32 个进程最终覆盖下标 $0$ 到 $31$。父进程等待直接子进程并按位或退出状态，形成从叶子向根汇总的结构。非根进程的输出不可见，但下面的命令可以跟踪全部进程，并统计执行 `correct` 输出的次数：

```sh
strace -f ./dist/beginners_rev "$1" 2>&1 | grep correct | wc -l
```

由于正确状态沿进程树从后向前汇总，爆破应从末位开始。设当前正在求下标 $i$，后缀 $i+1\ldots31$ 已经正确；当候选字符也正确时，统计值应为 $32-i$。官方求解器据此实现了一个简单的 oracle：

```python
import string
import subprocess

def oracle(candidate):
    proc = subprocess.Popen(
        ["./oracle.sh", candidate], stdout=subprocess.PIPE
    )
    output, _ = proc.communicate()
    return int(output.decode())

answer = ["*"] * 32
for i in range(31, -1, -1):
    for char in string.printable:
        answer[i] = char
        candidate = "".join(answer)
        if oracle(candidate) == 32 - i:
            break

print("".join(answer))
```

运行后得到：

```text
TSGCTF{y0u_kN0w_m@ny_g0od_t0015}
```

`is_correct` 内还有基于返回地址的反调试提示，直接逐条还原那段模 $367$ 的混淆运算并非必要；系统调用侧信道已经给出了更稳定的逐字符判定方法。

## 方法总结

本题的关键是把“每个进程只校验一个字符”和“`strace -f` 能越过标准输出重定向观察系统调用”结合起来。面对大量 `fork`、`dup2`、`wait` 的程序，应先画清进程树、下标生成方式与退出状态传播方向，再决定爆破顺序。这里必须从末位向前恢复，否则已知正确字符不能形成单调的计数反馈。反调试和冗长算术只是次要障碍，动态侧信道才是最短解法。
