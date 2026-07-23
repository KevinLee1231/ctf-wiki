# Philip 3

## 题目简述

第三关的 flag 被一个自编译的 `flag-vault` 进程保存在堆中。程序没有把 flag 连续存成普通字符串，而是将字符放进链表节点，需要结合源码、进程映射和堆转储还原。

## 解题过程

先从进程和 bash 历史定位目标：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_psaux
vol.py -f memory.lime --profile=<LinuxProfile> linux_bash
```

可见 `flag-vault` 的 PID 为 `9793`，历史中还保留了从 C 源码编译它的命令。恢复源码后可知：程序为每个字符 `malloc` 一个链表节点，节点同时保存字符和下一个节点指针。

列出该进程的内存映射，确定堆起始地址：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_proc_maps -p 9793
```

本镜像中的堆从：

```text
0x000055e46125e000
```

开始。转储对应映射：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_dump_map \
  -p 9793 -s 0x000055e46125e000 -D dump-map
```

根据源码中的结构体布局，从头节点开始读取字符字段并跟随 `next` 指针，按链表顺序拼接出：

```text
UMDCTF-{V0l$h311_isCool}
```

## 方法总结

内存中的逻辑顺序不一定等于物理连续顺序。源码给出了节点布局和遍历关系，进程映射给出地址基准，堆转储提供实际字节；三者结合后才能正确跟随指针。直接对镜像跑 `strings` 可能只看到零散字符，无法证明顺序。
