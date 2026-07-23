# UMDCTF2022 Kernel Infernal 1 Writeup

## 题目简述

题目给出 Ubuntu 20.04 的内核崩溃转储 `ubuntu20.04-5.4.0-99-generic-cloudimg-20220215.kdump`，询问内核事故发生在哪里。分析需要使用与转储版本匹配、带调试符号的 `vmlinux-5.4.0-99-generic`，然后从 panic 时的调用栈定位相关进程及其当前工作目录。

## 解题过程

使用 `crash` 同时加载符号文件和转储：

```bash
crash vmlinux-5.4.0-99-generic \
  ubuntu20.04-5.4.0-99-generic-cloudimg-20220215.kdump
```

先查看崩溃现场的回溯：

```text
crash> bt
```

回溯和任务信息将注意力指向一个 `bash` 进程，PID 为 `5206`。随后查看该进程打开的文件及路径信息：

```text
crash> files 5206
```

输出中的当前工作目录揭示了题目所问的“scene of the crime”，其关键目录名为：

```text
T0ta11yCR45H!!
```

按题目格式包裹后得到：

```text
UMDCTF{T0ta11yCR45H!!}
```

公开仓库没有附官方命令记录；[参赛者复盘](https://sutharnisarg.medium.com/umdctf-2022-write-ups-c45cdef017bb)确认了使用匹配内核符号加载 `.kdump`、从 `bt` 定位 PID `5206`，再通过进程文件信息查找现场路径的路线。正文已概括其有效信息。

## 方法总结

Linux kdump 题首先要保证 `vmlinux` 与转储内核版本严格匹配，否则结构体偏移和符号解析都会失真。定位路径类问题时，调用栈只负责找出相关任务；还要结合 PID 查询该任务的工作目录、根目录和已打开文件，才能把崩溃行为落到具体文件系统位置。
