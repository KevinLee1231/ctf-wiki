# week1签到

## 题目简述

附件是 32 位 Windows PE，flag 以明文形式保存在只读数据区中。程序虽然会提示输入一个数字，但获取 flag 不需要还原输入校验，只需检查二进制的静态字符串。

## 解题过程

先确认文件类型，再提取可打印字符串：

```bash
file 签到1.exe
strings -a 签到1.exe
```

`file` 显示它是 `PE32 executable, Intel i386`。字符串列表中可直接看到：

```text
welcome to re world!
plz input a number:
0xgame{w3lcom3_20_oXCtf!}
maybe you need ida or od
```

在 IDA 中也可以按 `Shift+F12` 打开 Strings 窗口，双击 flag 字符串查看其交叉引用。由于 flag 本身已经完整存在于文件中，无须继续动态调试。

## 方法总结

- 核心技巧：对未混淆的二进制先做静态字符串检查。
- 识别信号：程序体积小、无壳，题目定位为入门签到，数据段中存在完整 flag 格式。
- 复用要点：先用 `file` 和 `strings` 做低成本侦察；只有字符串被编码或运行时生成时，才需要进一步反编译。
