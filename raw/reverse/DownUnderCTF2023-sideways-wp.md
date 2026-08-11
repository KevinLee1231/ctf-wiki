# DownUnderCTF 2023 Sideways Writeup

## 题目简述

程序校验一个 26 字节 flag，每轮同时检查对称位置 `input[i]` 与 `input[25-i]`。检查函数在比较自定义 MD5 风格摘要之前，会先执行与两个字符乘积有关的循环；某一对错误时程序立即退出。

摘要本身不适合直接求逆，但“提前退出”与“乘积控制的循环次数”共同形成了指令计数侧信道，可以从外向内一次恢复两个字符。

## 解题过程

第 $i$ 轮使用字符 $a=\text{input}[i]$ 和 $b=\text{input}[25-i]$。Rust 发布版本中的 8 位乘法按模 256 回绕，因此循环次数由

$$
(a\times b)\bmod256
$$

决定。如果当前猜测错误，程序会在本轮结束；如果正确，则会进入下一轮。于是可以在下一对位置放入两组循环次数差异极大的哨兵字符：

```text
'O' * 'Q' mod 256 = 255
'0' * 'p' mod 256 = 0
```

对同一个当前猜测分别运行这两个输入，并用 Linux `perf` 统计用户态退休指令数：

```python
import re
import subprocess

def instruction_count(candidate):
    while True:
        proc = subprocess.run(
            ["perf", "stat", "-e", "instructions:u", "./sideways", candidate],
            capture_output=True,
            text=True,
        )
        match = re.search(r"\s*([\d,]+)\s+instructions:u", proc.stderr)
        if match:
            return int(match.group(1).replace(",", ""))
```

若当前字符对正确，执行流会到达下一轮，两个哨兵分别多执行 255 次和 0 次循环，实测指令差接近 145000；若当前字符对错误，程序提前退出，两次计数便没有这段稳定差异。枚举 `ascii_letters + digits + '{}_'`，选择

$$
\left|\,|c_1-c_2|-145000\,\right|
$$

最小的候选对即可。已知格式还能固定左侧的 `DUCTF{` 和最右侧的 `}`，减少最初几轮的搜索量。

逐轮保存已确认的对称字符，最终恢复：

```text
DUCTF{s1de_ch4nn3ld_af39c}
```

`perf` 可能偶尔报告 `<not counted>`，求解器应重试该次测量；系统噪声也会造成小幅浮动，所以比较与目标差值，而不是要求计数完全相等。

## 方法总结

本题不需要逆向摘要算法的碰撞或原像。决定性信息来自控制流：正确前缀会让程序继续执行，而下一轮可控循环把这个布尔事实放大为明显的指令数差。处理此类比较器时，应同时检查提前退出、数据相关循环和可测量的硬件计数器，而不只关注最终密码函数。
