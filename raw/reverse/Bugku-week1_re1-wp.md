# week1_re1

## 题目简述

题目是一个入门级 32 位 PE。程序没有复杂校验，flag 以明文形式硬编码在 `.rdata` 段中，直接字符串搜索即可获得。

## 解题过程

用 Ghidra 搜索 `flag`，或使用 `strings`：

```bash
strings re1.exe | grep "0xGame"
```

可以直接看到：

```text
Here is your flag :
0xGame{be9d9fee-7d45-4a3e-a105-802b3221665d}
```

## 方法总结

- 核心技巧：明文字符串搜索。
- 识别信号：入门 PE 中存在明显 `flag` 字符串且无二次校验时，先做字符串和交叉引用检查。
- 复用要点：简单题不要过度反编译，先用 `strings`、Ghidra Strings、交叉引用确认是否为真实输出。
