# TSGCTF2024 TSGDBinary

## 题目简述

附件并不是一个能独立完成验证的普通 ELF。`start.sh` 实际执行：

```sh
sudo gdb --nx -x ./tsgdbinary.py ./tsgdbinary
```

ELF 的 `main` 只把输入与 dummy 字符串比较：

```c
char flag[0x30] = "TSGCTF{dummy_plz_execute_start.sh}";
scanf("%47s", buf);
if (!memcmp(buf, flag, 0x30)) {
    puts("Correct!");
}
```

真正的检查由混淆 GDB Python 脚本完成。脚本会提取 ELF 内嵌字节、生成新的 GDB 脚本、修改内存与断点、调用二进制中的小函数，并在固定 RW/RWX 区重建机器码。目标是把 GDB 和 ELF 视为同一个联合状态机，逆出 48 字节 flag。

## 解题过程

### 1. 整理混淆 Python 的基础原语

`tsgdbinary.py` 用字符拼接隐藏命令和寄存器名，例如：

```python
wnket = chr(115) + chr(104) + chr(101) + chr(108) + chr(108) + ...
```

恢复后可得到 `shell rm ./kk`、`set`、`history` 等字符串。几个频繁调用的包装函数分别表示：

```text
koqw(reg, value)       set $reg = value
oqwk(dst, addr)        set $dst = *(unsigned char *)$addr
qneu(addr_reg)         把 $al 写回该地址
xec(addr, length)      从 inferior 读取内存
lop(bytes, key)        XOR 解密后写入临时脚本
ckp(addr)              设置静默断点
ehc(n)                 从 GDB history 文件执行第 n 条命令
```

history 的作用主要是把反复执行的 `continue` 等命令藏起来，并没有引入不可逆状态。

### 2. 提取三段内嵌脚本

主脚本从 ELF 映像的固定偏移读取三段数据并异或：

```text
base + 0x3080，长度 71， XOR 0xd7
base + 0x30e0，长度 305，XOR 0x3c
base + 0x3220，长度 270，XOR 0x48
```

恢复出的脚本依次完成：

1. 在 `0x6547ea867000` 映射 0x1000 字节 RWX 区，作为运行时机器码区。
2. 在 `0x72433a3c000` 映射 0x400 字节读写区，并把 0x30 字节用户输入复制过去。
3. 执行后续 Python/GDB 控制逻辑，逐字节变换输入并构造六张机器码查找表。

第二段脚本把输入源地址硬编码为 `0x7fffffffe460`。这个地址依赖具体环境；官方 WP 也确认它可能导致正确 flag 在其他系统上仍被判错。分析时应在 `scanf` 后读取实际 `buf` 地址并替换 `$src`，或直接把候选输入写到 `0x72433a3c000`，不能把运行失败误认为逆向结论错误。

### 3. 恢复逐字节 XOR 层

主脚本循环 0x30 次，每轮从输入工作区取一个字节，设置寄存器后跳入 ELF 中的简单算术函数，并把 `$al` 写回原位置。参与运算的另一个字节来自 ELF 内嵌数组。把 GDB 包装函数还原后，这一层只是逐字节 XOR；记录每轮使用的常量即可写成：

```python
stage1 = bytes(candidate[i] ^ xor_key[i] for i in range(0x30))
```

XOR 自反，因此最后恢复 flag 时使用同一组 `xor_key` 再异或一次即可。

### 4. 从六张机器码表逆查 64 位块

脚本随后处理六个 8 字节块。它把 ELF 中的加密字节复制到 RWX 区，并通过多轮 XOR 恢复 `tc0` 等 switch table。每张表都由大量重复分支组成，反汇编后形式为：

```asm
movabs rax, <candidate_u64>
cmp    qword ptr [rbp-0x10], rax
jne    next_case
movabs rax, <return_u64>
ret
```

因此每个表本质上是映射：

```text
candidate_u64 -> return_u64
```

在最终 0x30 字节比较前下断点，可分别 dump 六个期望返回值；也可以直接从生成 C 的最终比较缓冲区读取它们。对第 $i$ 张表，寻找满足 `mapping[candidate] == expected[i]` 的唯一候选，就得到逐字节 XOR 后的第 $i$ 个 8 字节块：

```python
stage1_blocks = []
for table, wanted in zip(tables, expected):
    reverse = {returned: candidate for candidate, returned in table.items()}
    stage1_blocks.append(reverse[wanted].to_bytes(8, "little"))

stage1 = b"".join(stage1_blocks)
flag = bytes(x ^ k for x, k in zip(stage1, xor_key))
```

恢复结果为：

```text
TSGCTF{0bfu5ca70r_can_al50_u53_gdb_and_b1nary}
```

## 方法总结

本题的难点是把调试器脚本也纳入逆向边界：GDB 负责解密子脚本、改写内存、恢复机器码并驱动 ELF 中的小函数，单看 `main` 会只看到 dummy。有效方法是先给混淆包装函数重新命名，再按“脚本提取层、逐字节 XOR 层、六个 64 位查找表、最终比较”分层记录输入输出。题目还存在硬编码栈地址的环境缺陷，复现时必须以实际 `buf` 地址修正，而不能假定官方地址在所有系统上成立。
