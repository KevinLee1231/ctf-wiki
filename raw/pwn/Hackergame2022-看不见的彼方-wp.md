# 看不见的彼方

## 题目简述

需要上传两个 Linux x86-64 程序。Alice 与 Bob 运行在不同的 `chroot` 根目录中：Alice 能读 `/secret`，但输出被丢弃；Bob 看不到 secret，只有它的标准输出会返回。网络和 ptrace 等系统调用被 seccomp 禁止，目标是让 Bob 原样输出 Alice 的随机 secret。

核心缺陷是 `chroot` 只改变路径解析根目录，没有隔离 PID、用户或内核 IPC 状态。两个进程仍在同一 PID 命名空间，并以相同 UID 运行，因此可以互发信号。

## 解题过程

### 通过信号握手发现 PID

两个 chroot 都没有可用的 `/proc`，但容器中的 PID 很小，可以枚举一个窄范围。Bob 先用 `sigaction` 注册握手信号处理器；Alice 稍等 Bob 初始化后，向 PID 2 至 19 发送 `SIGUSR1`。

启用 `SA_SIGINFO` 后，Bob 能从 `siginfo_t.si_pid` 取得发送者 PID，并回送 `SIGUSR2`。Alice 的 ACK 处理器同样从 `si_pid` 记录 Bob 的 PID：

```c
static volatile sig_atomic_t peer = -1;

static void remember_sender(int sig, siginfo_t *info, void *ctx) {
    peer = info->si_pid;
}

struct sigaction sa = {0};
sa.sa_sigaction = remember_sender;
sa.sa_flags = SA_SIGINFO;
sigemptyset(&sa.sa_mask);
sigaction(SIGUSR2, &sa, NULL);
```

Bob 的握手处理器只需执行异步信号安全的 `kill(info->si_pid, SIGUSR2)`。不要在处理器里使用 `printf`、`malloc` 等非异步信号安全函数。

### 用实时信号传输每个字节

普通信号不会排队：同一种信号在尚未处理时再次到达，多个实例可能合并。使用 `SIGRTMIN` 这类 POSIX 实时信号可保证同类信号按发送顺序排队，并且 `sigqueue` 能携带一个 `sigval`。

Alice 读取 secret 后逐字节发送：

```c
FILE *fp = fopen("/secret", "rb");
unsigned char secret[128];
size_t n = fread(secret, 1, sizeof(secret), fp);

for (size_t i = 0; i < n; i++) {
    union sigval value = {.sival_int = secret[i]};
    while (sigqueue(peer, SIGRTMIN, value) == -1 && errno == EAGAIN) {
        sched_yield();
    }
}
```

Bob 为 `SIGRTMIN` 注册独立处理器，从 `si_value.sival_int` 取出低 8 位并直接写到标准输出：

```c
static void receive_byte(int sig, siginfo_t *info, void *ctx) {
    unsigned char ch = (unsigned char)info->si_value.sival_int;
    write(STDOUT_FILENO, &ch, 1);
}
```

题目 secret 由 `secrets.token_hex(32)` 生成，正好是 64 个十六进制字符，没有换行。Bob 输出 64 个数据字节后继续等待即可，服务端会对 `stdout.strip()` 与原 secret 做比较。

### 编译与提交

chroot 环境不保证包含动态链接库，所以两个程序都应静态链接，并尽量减小体积：

```bash
musl-gcc -static -Os alice.c -o alice
musl-gcc -static -Os bob.c -o bob
strip alice bob
```

分别 Base64 编码，再按服务端要求用 `@` 连接后提交。两者合计须小于 10 MiB，并在 10 秒内完成。

官方题解还给出过用普通 `SIGUSR2` 加短暂 `usleep` 的版本，但它依赖调度时序，连续信号可能丢失；实时信号方案不需要用延时猜测接收速度，可靠性更高。题解中的动漫截图只用于文案氛围，与 IPC 机制无关，故未保留。

## 方法总结

文件系统视图不同不等于进程完全隔离。`chroot` 没有创建 PID、IPC 或用户命名空间；相同 UID 的进程仍可通过信号共享信息。评估沙箱时，应逐项确认 mount、PID、IPC、network、user 等命名空间和系统调用策略，而不能把 `chroot` 当作容器。利用信号传数据时则应优先使用可排队、保序并携带值的实时信号。
