# DownUnderCTF 2020 - Hide-And-Seek

## 题目简述

题目提供一台可运行的 VirtualBox Linux 虚拟机，系统中 `/opt/malware` 存在可疑 Python 字节码和 `libprocesshider.so`。恶意进程明明在执行，却不会出现在 `ps`、`top` 等常规进程列表中。决定性问题不是分析网络投递，而是绕过用户态进程隐藏，从虚拟机内存或已删除文件中恢复 Python 进程的状态与 flag。

## 解题过程

### 确认进程隐藏机制

检查 `/opt` 后可以找到名为 `mother.cpython-38.pyc` 的程序。常规进程查看工具没有显示它，但对 `libprocesshider.so` 执行 `strings` 能看到 `python3` 字符串。这符合典型的 `LD_PRELOAD` 进程隐藏库：它拦截目录枚举等 libc 调用，过滤命令行中含 `python3` 的进程。内核仍然保存真实任务结构，因此离线内存取证不会受该过滤层影响。

### 导出虚拟机内存

先为该虚拟机内核构建匹配的 Volatility Linux profile，然后从宿主机导出 VirtualBox core：

```bash
VBoxManage debugvm "hide-and-seek" dumpvmcore --filename test.elf
objdump -h test.elf | grep -E -w '(Idx|load1)'
```

官方环境中 RAM 段大小为 `0x80000000`，偏移为 `0x2598`。据此从 ELF core 中切出裸内存：

```bash
size=0x80000000
off=0x00002598
head -c $((size + off)) test.elf | tail -c +$((off + 1)) > test.raw
```

偏移和大小应以自己的 `objdump -h` 输出为准，不能无条件照抄。

### 定位隐藏 Python 进程

把生成的 profile 放入 Volatility 的 Linux overlay 目录后，使用内存中的任务结构列出进程：

```bash
volatility \
  --profile=LinuxUbuntu_5_4_0-45-generic_profilex64 \
  -f test.raw linux_psaux | grep python3
```

此时可以看到被用户态工具过滤掉的命令行：

```text
python3 /opt/malware/mother.cpython-38.pyc
```

记下 PID，回到仍在运行的虚拟机，用 `pyrasite-shell` 注入该 Python 进程的解释器上下文：

```bash
pyrasite-shell <PID>
```

在交互环境中枚举全局对象即可发现 `flag` 变量：

```python
dir()
print(flag)
```

输出为：

```text
DUCTF{w3_R_4bs0lute_pr3d4t0rs}
```

### 备用恢复路径

如果不做运行态注入，也可以把虚拟磁盘通过 `vboximg-mount` 暴露出来，再用 `testdisk` 恢复被删除的 `.pyc`。用支持 Python 3.8 字节码的反编译器还原后，可以看到程序通过 `exec` 组装 flag。该路径同样证明题目的核心是从被隐藏或删除的运行材料中恢复事实，而不是信任虚拟机内的 `ps` 输出。

## 方法总结

- 核心技巧：识别 `LD_PRELOAD` 风格的进程隐藏，并转向虚拟机内存取证或删除文件恢复。
- 识别信号：程序确实运行但 `ps` 不显示、可疑共享库含被过滤的进程名、题目允许导出 VM core 时，应从内核任务结构或进程解释器状态取证。
- 复用要点：Volatility profile 必须与目标内核准确匹配；从 VirtualBox core 切 RAM 时要使用实际 section 偏移；Python 进程可通过注入解释器直接读取全局变量，磁盘恢复则是可独立验证的备用证据链。
