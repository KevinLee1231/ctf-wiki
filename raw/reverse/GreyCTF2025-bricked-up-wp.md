# Bricked Up

## 题目简述

这是一道运行在比赛徽章上的方块游戏题。选手需要从 MicroPython `.mpy` 字节码中恢复隐藏按键序列，触发分数倍率，再在实体设备上达到目标分数。决定性障碍是恢复 Python 字节码行为，因此归入 reverse，而不是仅按原目录的 hardware 标签归档。

## 解题过程

徽章上的文件使用 `.mpy` 格式。先识别其 MicroPython 版本，把字节码头修正为 `mpy-tool.py` 支持的对应头部，再反汇编：

```bash
python mpy-tool.py -d challenge.mpy
```

在反汇编的按键状态机中可以读出一组经典 Konami Code：

```text
UP UP DOWN DOWN LEFT RIGHT LEFT RIGHT B A
```

按顺序输入后，游戏打开 $1000\times$ 分数倍率。随后正常进行方块游戏，把总分提升到至少 10000；可以一次消四行，也可以连续清除 10 条单行，连击实现会使多次清行更快达到阈值。

本地逻辑先显示中间提示 `grey{go_do_this_on_stage}`，要求到现场徽章上复现。同样操作在舞台设备上输出正式 flag：

```text
grey{c4ts_h4v3_30_1iv3s!}
```

## 方法总结

`.mpy` 不是普通 CPython `.pyc`，不能只改扩展名；必须先匹配 MicroPython 字节码版本和头部，再用专用工具反汇编。恢复状态机后，应把隐藏序列、倍率条件和得分阈值分别验证。此题最后依赖实体徽章，题解应明确区分本地占位提示与现场正式结果，避免把中间字符串误当 flag。
