# BSidesAlgiers2025 - Excavator

## 题目简述

题目给出一个在事故响应中取得、部分损坏的进程 core dump。配套的 `stager.c` 表明程序会从 ZIP 中取出 `config.bin`，再使用内存中的 `struct state { s[256]; i; j; }` 解密；这个结构正是已经完成 KSA 的 RC4 状态。目标是同时从 core 中找出压缩数据和 RC4 状态，还原恶意程序配置。

附件 `excavator.zip` 解压后包含 `core.522044` 与 `stager.c`，仓库的 `solution/solve.py` 给出了自动恢复实现。

## 解题过程

RC4 的 `S` 盒必然是 $0$ 到 $255$ 的一个排列。因此可以在 core 中滑动 256 字节窗口，筛出“256 个值互不重复且集合完整”的位置，并把紧随其后的两个字节解释为 `i`、`j`。本题唯一有效状态位于偏移 `540472`，且 `i=j=0`。

损坏的 ZIP 仍留下了 DEFLATE 数据。官方脚本从每个 core 偏移取最多 256 字节，分别尝试 zlib 包装流和裸 DEFLATE；把成功解出的片段逐一送入候选 RC4 状态，只保留全可打印明文：

```python
def rc4_apply(s, i, j, data):
    s = s[:]
    out = bytearray()
    for value in data:
        i = (i + 1) & 0xff
        j = (j + s[i]) & 0xff
        s[i], s[j] = s[j], s[i]
        out.append(value ^ s[(s[i] + s[j]) & 0xff])
    return bytes(out)
```

在题目目录中解压附件后运行：

```bash
python3 solution/solve.py core.522044
```

本地复核得到：

```text
Found 1 RC4 state candidates
Found 716 decompression hits
RC4 state offset: 540472, i=0, j=0
Deflate stream offset: 14832, length=53
Plaintext:
{!!!Y0u_D1d_I7_l1k3_4n_3xC4vat0r!_w3eeell_d0ooone!!!}
```

恢复值本身从左花括号开始，按比赛统一前缀补上 `shellmates`，最终 flag 为：

`shellmates{!!!Y0u_D1d_I7_l1k3_4n_3xC4vat0r!_w3eeell_d0ooone!!!}`

这里不需要假设字符串格式：自动脚本输出、`stager.c` 的 RC4 状态结构与公开的 [Excavator 复盘](https://medium.com/@yanisderiche22/excavator-a9d55fab80af) 给出的最终结果三者一致。

## 方法总结

- 对损坏的 core 不应只跑 `strings`；压缩片段和运行中密码状态往往仍能独立雕取。
- RC4 的 256 字节排列是强结构特征，找到完整 `S` 盒后可以从当前 `i/j` 继续 PRGA，无需恢复原始密钥。
- 用“可解压片段 × 密码状态候选 × 可打印性筛选”缩小组合空间，既能自动化恢复，也保留了偏移、长度和状态值作为可复核证据。
