# unstoppable

## 题目简述

程序内置 5005 台五状态、二符号图灵机，要求提交其中全部 2703 台会停机的机器编号。一般停机问题不可判定，但这里状态数和符号数固定，并且五状态 Busy Beaver 的精确步数界已经得到证明：任何从空白纸带出发且最终会停机的五状态二符号机，都不会运行超过 $47\,176\,870$ 步。

[Coq-BB5](https://github.com/ccz181078/Coq-BB5) 仓库给出了结论 $BB(5)=47\,176\,870$ 的形式化验证材料。这个外部结论在本题中的直接含义是：模拟到该步数仍未进入停机态的机器，可以确定为不停机，而不只是“暂时没有停”。

## 解题过程

### 还原输入检查与 VM 格式

附件是 x86-64 PIE C++ 程序，开启 NX、无 Canary、Partial RELRO，并经过 OLLVM 控制流平坦化、虚假控制流和指令替换。输入逻辑最终读取 2703 个整数，每个编号必须满足：

```text
0 <= index < 5005
```

真正需要恢复的是 VM 表，而不是完整清理所有混淆。每台 VM 占 30 字节：

$$
5\text{ states}\times2\text{ symbols}\times3\text{ bytes}=30\text{ bytes}.
$$

对当前状态 $s$ 和当前纸带位 $v$，转移项位置为：

```c
op = vm + 3 * ((s << 1) | v);
```

三个字节依次表示：

```text
op[0]：写入的符号
op[1]：移动方向，非零向右，否则向左
op[2]：下一状态，25 表示停机
```

初始纸带全为 0，状态为 0，读写头位于中间。纸带两端按需扩展，因此不能用固定长度数组后直接越界。

### 方案一：导出全部 VM 后精确模拟

第一层混淆函数负责输入和调度，可使用 [Hrtng](https://github.com/KasperskyLab/hrtng) 辅助恢复。Hrtng 是 IDA 插件，包含 OLLVM 控制流平坦化、虚假控制流等反混淆能力；它只是加快定位数据表，解题所需的 VM 格式和停机判据仍应由代码验证。

导出 `opcode[5005][30]` 后，用原生代码并行模拟。核心逻辑如下：

```cpp
constexpr uint64_t BB5 = 47176870;

#pragma omp parallel for
for (int id = 0; id < 5005; ++id) {
    std::vector<uint8_t> tape(1024, 0);
    uint64_t head = 511;
    uint8_t state = 0;

    for (uint64_t step = 0; step < BB5; ++step) {
        uint8_t *op = opcode[id] + 3 * ((state << 1) | tape[head]);
        tape[head] = op[0];

        if (op[1]) {
            if (++head == tape.size())
                tape.resize(tape.size() * 2, 0);
        } else {
            if (head == 0) {
                size_t old = tape.size();
                tape.insert(tape.begin(), old, 0);
                head += old;
            }
            --head;
        }

        state = op[2];
        if (state == 25) {
            halting[id] = true;
            break;
        }
    }
}
```

步数上界应按“最多执行 $47\,176\,870$ 次转移”实现，避免循环边界少模拟一次。完成后收集 `halting[id] == true` 的编号，排序并检查数量恰好为 2703，再按程序要求提交。没有必要把 2703 个数字全部写入 WP；它们应由脚本从附件确定性生成。

### 方案二：补丁化原程序逐编号测试

如果不想完整还原第二个重混淆函数，可以修改程序，使每次只读取一个编号，并绕过最终批量哈希或答案检查。原程序内部已经有 VM 执行器：会停机的编号将正常返回，不停机的编号则持续运行。对 $0$ 到 $5004$ 并行启动补丁后的程序，即可分类。

官方思路使用 `multiprocessing` 加墙钟超时，但单纯把 40 秒当作停机界限并不严谨：机器负载变化可能把接近上界的可停机实例误判为不停机。更可靠的补丁是在执行器中加入精确的 $47\,176\,870$ 步计数，到界后返回“不停机”；若只能使用进程超时，则应逐步增加阈值，并对超时样本在低负载下复查，直到结果稳定且停机数量为 2703。

一个基本的并行外壳为：

```python
def test_number(n):
    proc = subprocess.Popen(
        ["./unstoppable_patched"],
        stdin=subprocess.PIPE,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )
    try:
        proc.communicate(f"{n}\n".encode(), timeout=timeout)
        return n
    except subprocess.TimeoutExpired:
        proc.kill()
        proc.communicate()
        return None
```

最后同样要排序结果、去重并验证数量，不能直接使用并行任务的完成顺序作为提交顺序。

## 方法总结

本题把不可判定的停机问题限制成了已有严格上界的有限分类问题。逆向工作的最小目标是恢复 30 字节转移表和输入契约，不必为了“反混淆完整”而清理所有函数。拿到 VM 后，以 Coq-BB5 的精确步数界进行确定性模拟是最可靠方案；补丁加进程超时可以省逆向工作，但必须处理墙钟阈值带来的误判风险。
