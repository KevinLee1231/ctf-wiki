# Neon Deceit

## 题目简述

程序无论是否附加调试器，表面上都只输出 `hello world`。ELF 的 `.text` 中塞入了约 200 KiB 的 NOP，大量 libc 导入名也被故意用于错误语义，IDA 还会因为对 `verrx` 的函数属性判断而提前截断真实函数。

真正逻辑包含名称型反调试、伪造退出调用和一个 $21\times51$ 的迷宫。用户需提交 40 个十六进制字符；程序将其解析成 20 字节，再把每个字节拆成四组 2 bit，合计生成 80 步移动。

仓库 checker 给出的精确目标是：

```text
R3CTF{ffffffffffffd7d5556aa97d7ffffffffffffd57}
```

## 解题过程

### 修正反编译边界

入口附近看似存在正常的 `_exit`，但继续观察实际 `ret` 与交叉引用会发现，这些导入名只是混淆。`main` 末端仍有：

```asm
cmp     [rbp+argc], 7
jnz     fake_path
call    real_logic
```

因此要以程序名加六个参数启动，或在分析副本中临时翻转该跳转。真实大函数位于 `main` 之前；需要取消 IDA 因 `verrx` 语义造成的截断，重新定义函数范围后再反编译。

程序还反复处理密文：

```text
=>866>83>;).(;9?6.(;9?(;>;(?h(h
```

逐字节异或 `0x5a` 后得到若干调试器名称，说明这里按进程名检测调试环境。调试时可修改该密文或绕过相应条件分支。其余看似 `cimag`、`vfwprintf`、`realloc` 的调用也不能按导入名理解，应结合参数、返回值检查和调用后的控制流重新命名。

### 恢复迷宫路径

绕过假退出分支后，程序在栈上生成迷宫。入口位于左上可通行区域，终点检查坐标为：

```text
row = 19
col = 50
```

按 `w/a/s/d` 记方向，从入口到出口的一条有效路径是：

```text
dddddddddddddddddddddddddssddssssssssaaaaaassddssddddddddddddddddddddddddddssssd
```

路径长度正好为 80。程序还维护已访问表，越界、撞上 `#` 或再次进入已访问位置都会失败，所以不能用无意义往返补长度。

### 把 80 步编码为 20 字节

输入必须是 40 个十六进制字符，解析后得到 20 字节。每个字节按从高位到低位的顺序拆为四组 2 bit，映射为：

```text
00 -> w
01 -> s
10 -> a
11 -> d
```

编码时每四步合成一字节：

```python
code = {"w": 0, "s": 1, "a": 2, "d": 3}
out = bytearray()

for i in range(0, len(path), 4):
    value = 0
    for direction in path[i:i + 4]:
        value = (value << 2) | code[direction]
    out.append(value)

answer = out.hex()
```

对上述路径编码得到：

```text
ffffffffffffd7d5556aa97d7ffffffffffffd57
```

程序最终使用 `R3CTF{%s}` 输出，因此 flag 为：

```text
R3CTF{ffffffffffffd7d5556aa97d7ffffffffffffd57}
```

## 方法总结

本题先用 NOP 填充、错误导入名、IDA 函数截断和名称型反调试制造分析噪声，实际校验则是清晰的“十六进制输入 → 2 bit 方向 → 迷宫行走”。关键是以真实控制流和数据流为准，不把动态符号名称当作函数语义；恢复迷宫后，剩余工作只是固定映射编码。

[详细逆向记录](https://astralprisma.github.io/2025/07/07/r3_25_neon/) 展示了函数边界修复、迷宫反汇编和坐标校验；正文已保留完整路径、2 bit 映射、编码方法与最终结果。
