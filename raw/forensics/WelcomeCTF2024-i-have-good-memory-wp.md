# I Have Good Memory

## 题目简述

题目提供 Windows 内存转储，并提示进程内存会保留可执行文件及其命令行参数。目标是使用 Volatility 检查进程列表与详细参数，从历史进程中找到 flag。

## 解题过程

先让 Volatility 识别并列出 Windows 进程。仓库中的官方 `solve.txt` 明确提示对 `windows.pslist` 或 `windows.pstree` 使用 verbose 模式；在常见 CLI 中，`-v` 是全局选项，应放在插件名之前：

```bash
python3 vol.py -v -f memory.raw windows.pslist
```

也可以使用进程树观察父子关系：

```bash
python3 vol.py -v -f memory.raw windows.pstree
```

需要注意，不同 Volatility 版本对 verbose 输出的处理并不一致：部分 Volatility 3 版本的 `-v` 只增加框架日志，并不会给进程表增加命令行列。遇到这种情况，应先用 `pslist` 或 `pstree` 确定可疑进程，再直接调用命令行插件：

```bash
python3 vol.py -f memory.raw windows.cmdline
```

在输出中搜索 `grey{`，或重点检查可疑解释器、终端和临时进程的参数。即使进程已经结束，只要相关内存页尚未被覆盖，参数字符串仍可能存在。由进程参数可恢复：

```text
grey{i_s3e_wH@t_y0u_d1D_tH3rE}
```

## 方法总结

内存取证中的进程名只是入口，命令行、环境变量、打开句柄与父子关系往往更有价值。本题的决定性线索是进程参数；同时要区分官方解法表达的分析意图和具体工具版本的参数语义，避免机械照抄一条在当前版本中无效的命令。
