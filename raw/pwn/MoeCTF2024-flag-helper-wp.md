# flag_helper

## 题目简述

程序把 `open`、匿名 `mmap`、`read` 和 `write` 的关键参数交给用户输入。目标不是劫持控制流，而是正确组合 Linux 系统调用参数，把 `/flag` 读入可写内存并输出。

## 解题过程

核心流程可以简化为：

```c
open("/dev/random", O_RDONLY);
open("/dev/urandom", O_RDONLY);
int flag_fd = open(file, open_flags);
open("/dev/zero", O_RDONLY);

char *buf = mmap(NULL, 0x50, prot, map_flags, -1, 0);
read(fd, buf, 0x50);
write(STDOUT_FILENO, buf, 0x50);
```

各项输入应为：

- 文件名：`/flag`；
- `open_flags = O_RDONLY = 0`；
- `prot = PROT_READ | PROT_WRITE = 1 | 2 = 3`；
- `map_flags = MAP_PRIVATE | MAP_ANONYMOUS = 2 | 32 = 34`；
- `fd = 5`。

文件描述符从 0 开始分配。进程已有标准输入、输出、错误三个描述符，随后打开 `/dev/random` 和 `/dev/urandom`，所以 `/flag` 获得的描述符是 5；后打开的 `/dev/zero` 不会改变它。

最终依次输入：

```text
4
/flag
0
3
34
5
```

其中开头的 `4` 是程序菜单选项。题目提示 `mmap` 的 `fd` 为 `-1`，正是在说明这里使用的是匿名映射，而不是文件映射。

## 方法总结

本题训练的是把头文件宏还原为数值并追踪文件描述符分配。`MAP_ANONYMOUS` 与 `fd=-1` 共同表示分配空白内存；由于后续 `read` 会写入该区域，权限必须同时包含 `PROT_READ` 和 `PROT_WRITE`。
