# I Have Good Memory

## 题目简述

题目原始附件是一份 Windows 内存转储，提示可执行文件及其启动参数会留在内存中。仓库未包含转储本体，只保留了外部下载说明、官方解法提示和最终 flag；可复现主线是在进程列表中定位可疑进程，再检查其命令行参数。

## 解题过程

先用 Volatility 枚举进程与父子关系：

```bash
vol -f memory.raw windows.pslist
vol -f memory.raw windows.pstree
```

官方简解写的是对 `pslist` 或 `pstree` 使用 verbose 选项。不同 Volatility 版本对 `-v` 的含义并不完全相同；在 Volatility 3 中，它主要控制框架日志详细度，因此更稳妥的做法是继续调用命令行插件：

```bash
vol -f memory.raw windows.cmdline
```

在进程树中找到异常启动项后，检查其完整参数，flag 就作为命令行参数残留在转储里。仓库记录的结果为：

```text
grey{i_s3e_wH@t_y0u_d1D_tH3rE}
```

由于公开仓库没有保存内存镜像，当前只能复现插件选择与证据定位方法，不能核验具体 PID 或进程名；正文没有补造这些缺失字段。

## 方法总结

内存取证题若强调“执行参数”，只列进程名通常不够，应把 `pslist`/`pstree` 的结构定位与 `cmdline` 的参数恢复组合起来。还要注意 Volatility 2、Volatility 3 以及不同封装命令的参数差异，不应把日志 verbosity 误当作必然会显示命令行字段。
