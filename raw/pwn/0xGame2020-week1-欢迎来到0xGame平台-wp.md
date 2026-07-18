# week1欢迎来到0xGame平台

## 题目简述

这是一道用于熟悉 Pwn 远程交互流程的签到题。程序完成标准输入输出初始化后，直接执行 `system("/bin/sh")`，因此连接服务即可进入 shell，不需要构造内存破坏漏洞。

关键逻辑如下：

```c
int main()
{
    my_init();
    puts("WelCome_To_0xCTF");
    system("/bin/sh");
}
```

## 解题过程

使用 `nc` 连接题目给出的主机和端口：

```bash
nc <HOST> <PORT>
```

看到欢迎信息后，标准输入已经连接到 `/bin/sh`。先查看当前目录中的文件，再读取 flag：

```sh
pwd
ls -la
cat flag
```

若 flag 不在当前目录，可通过 `find / -name 'flag*' 2>/dev/null` 确认位置后再读取。

## 方法总结

- 核心技巧：识别程序主动启动的 shell，直接进行命令交互。
- 识别信号：源码中存在无条件执行的 `system("/bin/sh")`。
- 复用要点：拿到 shell 后先确认工作目录和文件名，不要预设 flag 的绝对路径。
