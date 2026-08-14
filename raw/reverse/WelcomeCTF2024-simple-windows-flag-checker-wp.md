# Simple Windows Flag Checker

## 题目简述

附件是一个 Windows PE flag 检查程序。flag 以连续明文字符串形式编译进二进制，没有加密或运行时拼接，目标是从静态字符串中直接恢复它。

## 解题过程

先对程序提取可打印字符串并筛选赛事前缀：

```bash
strings program.exe | grep 'grey{'
```

也可以在十六进制编辑器或反编译器的 Strings 窗口中搜索 `grey{`。程序中直接出现：

```text
grey{str1ngs_t3lls_y0u_4l0t_4b0ut_pr0grams}
```

不需要运行未知 PE，也无需进入调试器；静态字符串已经是检查逻辑所使用的完整答案。

## 方法总结

逆向的第一步应是低成本静态侦察。硬编码凭据、路径、错误信息与 flag 常会留在字符串表中；只有字符串不足以解释或恢复结果时，才需要继续反汇编与动态调试。
