# TSGCTF2024 Warmup SQLite

## 题目简述

附件 `dump` 是 SQLite 对隐藏 SQL 执行 `EXPLAIN` 后得到的 VDBE 指令列表，其中参数位置被替换为 `~~Your input is filled here~~`。需要从 opcode 控制流还原 SQL 对 64 字符输入所做的变换，再反推出能通过检查的字符串。

`dump` 中有两段明显的协程/临时表循环：第一段使用 `substr` 逐字符拆分输入并维护索引；第二段在每个字符上反复执行 `Multiply`、`Add` 与 `Remainder`。

## 解题过程

第一段关键指令为：

```text
24 Function   ... substr(3)
26 Function   ... substr(2)
28 Add
```

它对应递归 CTE：每轮取 `rest` 的第一个字符作为 `input`，把 `substr(rest, 2)` 作为新的剩余串，并令索引加一。

第二段的运算指令和结尾常量为：

```text
57 Multiply
58 Add
59 Remainder
62 Add

93 Integer 10
94 Integer 7
95 Integer 2
96 Integer 256
```

结合第 55 行 `Ge` 的循环条件，可还原为对每个字符码执行 10 次：

$$x \leftarrow (7x+2)\bmod 256$$

最后仅保留 `iter = 10` 的记录并按原字符索引排序。完整 SQL 语义可概括为：

```sql
WITH RECURSIVE
split(input, rest, idx) AS (
  VALUES('', ?, -1)
  UNION ALL
  SELECT substr(rest, 1, 1), substr(rest, 2), idx + 1
  FROM split WHERE rest <> ''
),
tr(val, idx, iter) AS (
  SELECT unicode(input), idx, 0 FROM split WHERE input <> ''
  UNION ALL
  SELECT (val * 7 + 2) % 256, idx, iter + 1
  FROM tr WHERE iter < 10
)
SELECT * FROM tr WHERE iter = 10 ORDER BY idx;
```

因为 $\gcd(7,256)=1$，7 在模 256 下可逆，且：

$$7^{-1}\equiv183\pmod{256}$$

单轮逆变换为：

$$x\leftarrow183(x-2)\bmod256$$

对服务端的 64 个结果执行 10 轮逆变换：

```python
res = [
    100, 115, 39, 99, 100, 54, 27, 115, 69, 220, 69, 99, 100, 191,
    56, 161, 131, 11, 101, 162, 191, 54, 130, 175, 205, 191, 222, 101,
    162, 116, 147, 191, 55, 24, 69, 130, 69, 191, 252, 101, 102, 101,
    252, 189, 82, 116, 41, 147, 161, 147, 132, 101, 162, 82, 191, 220,
    9, 205, 9, 100, 191, 38, 68, 253,
]

inverse = pow(7, -1, 256)
for _ in range(10):
    res = [((x - 2) * inverse) % 256 for x in res]

print(''.join(map(chr, res)))
```

输出为：

```text
TSGCTF{SELECT_hacker_FROM_nerds_WHERE_level="disaster"_LIMIT_64}
```

## 方法总结

本题考查从 SQLite VDBE opcode 恢复高级查询语义。先通过 `InitCoroutine/Yield`、临时表和 `substr` 识别逐字符递归，再从常量寄存器与算术 opcode 还原仿射变换。由于乘数 7 在模 256 下可逆，逐轮应用逆函数即可恢复输入。遇到类似 SQL/字节码题时，先划分循环与数据流，再把寄存器操作提升为简洁公式，通常比逐条模拟全部虚拟机状态更直接。
