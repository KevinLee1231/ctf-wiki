# EVM - The Basics

## 题目简述

题目给出一小段 EVM runtime bytecode，要求找到能让执行流跳过 `SELFDESTRUCT` 并落到 `STOP` 的交易金额。

## 解题过程

逐条反汇编附件：

```text
PUSH0
CALLVALUE
PUSH2 0x1337
MUL
PUSH6 0xfdc29ff358a3
EQ
PUSH1 0x12
JUMPI
SELFDESTRUCT
JUMPDEST
STOP
```

要让 `JUMPI` 成立，必须满足：

$$msg.value\times0x1337=0xfdc29ff358a3.$$

整数相除得到：

```text
msg.value = 0xd34db33f5
```

所以提交：

```text
n00bz{0xd34db33f5}
```

## 方法总结

短 EVM 字节码无需部署即可静态求解。应直接以附件字节 `0xfdc29ff358a3` 为准；官方说明中的个别抄写值不一致，不能替代原始 runtime 证据。
