# ezWin variables

## 题目简述

题目给出 Windows 10 22H2 内存镜像 `win10_22h2_19045.2486.vmem`，要求使用 Volatility 3 从进程环境变量中寻找第一段 flag。该题与 `ezWin auth`、`ezWin 7zip` 共用同一镜像，但取证目标彼此独立。

## 解题过程

先确认 Volatility 能识别镜像，再枚举进程环境变量：

```sh
python3 vol.py -f win10_22h2_19045.2486.vmem windows.info
python3 vol.py -f win10_22h2_19045.2486.vmem windows.envars.Envars
```

`windows.envars.Envars` 会列出进程、PID、环境变量名和值。输出较多，可直接按 flag 前缀过滤：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.envars.Envars | grep -i hgame
```

结果中直接出现：

```text
hgame{2109fbfd-a951-4cc3-b56e-f0832eb303e1}
```

截图中的终端输出仅承载文本信息，因此已完整转写为命令和结果，不再保留图片。

## 方法总结

环境变量会随进程地址空间一起留在内存中，常包含临时配置、令牌或题目刻意放置的线索。处理 Windows 内存镜像时，先用 `windows.info` 确认符号和系统版本，再选择针对性的插件；本题无需做全盘字符串扫描，`windows.envars` 能提供更明确的进程上下文。
