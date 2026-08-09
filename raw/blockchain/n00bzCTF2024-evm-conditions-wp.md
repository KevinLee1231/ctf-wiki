# EVM - Conditions

## 题目简述

附件是 EVM runtime bytecode。需要手工跟踪栈与内存，求出使条件跳转到 `STOP`、避开 `SELFDESTRUCT` 的 `msg.value`，最后以十六进制提交。

## 解题过程

前半段把四个中间值写入内存：

```text
0x258  <- 0x0f * 0x70          = 0x690
0x90   <- 0x96 / 0x05          = 0x1e
0xfffa <- 0x09 ** 0x07         = 0x48fb79
0xbfabf<- 0x0539 XOR 0x26aa    = 0x2393
```

随后加载并组合这些值：

```text
3 * 0x48fb79 + 0x2393 = 0xdb15fe
4 * 0x690 + 0x1e      = 0x1a5e = 6750
```

`EQ` 与 `JUMPI` 要求：

$$0xdb15fe=msg.value+0x1a5e.$$

因此：

```text
msg.value = 0xdb15fe - 0x1a5e = 0xdafba0
```

flag 为：

```text
n00bz{0xdafba0}
```

## 方法总结

EVM 逆向最重要的是确认操作数出栈顺序，并区分内存偏移和值。将每个 `MSTORE` 结果单独记录后，最后的条件会化为普通整数方程。
