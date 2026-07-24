# Transportation Cipher

## 题目简述

附件是一份四段航程记录：

```text
FLORENCIA --> GISBORNE
MARACAIBO --> LANGARA
NADOR --> SULE
SAN ANTONIO --> COATEPEQUE
```

题目强调航班与旅行地点，并说明答案本身不是标准 flag 格式。决定性线索是机场通常使用三个字母的 IATA 代码。

## 解题过程

依次查询每个地名对应的机场代码：

```text
Florencia   -> FLA
Gisborne    -> GIS
Maracaibo   -> MAR
Langara     -> YLA
Nador       -> NDR
Sule        -> ULE
San Antonio -> SAT
Coatepeque  -> CTF
```

严格按题面顺序拼接：

```text
FLA GIS MAR YLA NDR ULE SAT CTF
FLAGISMARYLANDRULESATCTF
```

其中开头的 `FLAGIS` 和结尾的 `ATCTF` 是语法框架，中间的正文为：

```text
MARYLANDRULES
```

README 要求将找到的整段答案转为小写并放入 `UMDCTF-{}`。结合完整拼接结果，最终 flag 为：

```text
UMDCTF-{marylandrulesatctf}
```

其 SHA-256 与官方摘要 `f9c3e7f9d451bcbdf11121089d626ab5c30477d37756bbfbee7f0915abcd5d98` 一致。

## 方法总结

这是一道以地理实体为载体的编码题。航程中的“起点和终点”不是为了计算路线，而是为了提供八个机场代码；必须保留原顺序，才能自然读出 `FLAG IS ... AT CTF`。拼接后仍应遵循题目对大小写和 flag 外壳的明确要求。
