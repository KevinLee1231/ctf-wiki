# 不太分布式的软总线

## 题目简述

提交的 ELF 以 `nobody` 用户运行，通过环境变量指定的 Unix socket 接入题目自建的 D-Bus system bus。服务拥有 well-known name `cn.edu.ustc.lug.hack.FlagService`，对象路径为 `/cn/edu/ustc/lug/hack/FlagService`，接口同名，暴露三个方法：

```xml
<method name="GetFlag1"><arg type="s" direction="in"/></method>
<method name="GetFlag2"><arg type="h" direction="in"/></method>
<method name="GetFlag3"/>
```

第一问是正确发起方法调用；第二问要求传递不落盘的 Unix 文件描述符；第三问服务通过 D-Bus sender 查询调用者 PID，再检查 `/proc/<pid>/comm` 必须为 `getflag3`。决定性障碍是 Unix IPC、FD 传递及不可靠的进程身份判定，不是云控制面，因此按运行时权限边界归入 `pwn`。

## 解题过程

### 第一问：还原 D-Bus 调用四元组

服务端只在 `GetFlag1` 收到字符串 `Please give me flag1` 时返回 flag。环境已提供 `gdbus`，脚本直接调用即可：

```bash
gdbus call --system \
  --dest cn.edu.ustc.lug.hack.FlagService \
  --object-path /cn/edu/ustc/lug/hack/FlagService \
  --method cn.edu.ustc.lug.hack.FlagService.GetFlag1 \
  "Please give me flag1"
```

这里必须同时正确给出 bus 类型、destination、object path、interface/method 与 GVariant 参数类型，不能只知道方法名。

### 第二问：用 memfd 传匿名文件

D-Bus 的类型 `h` 不是直接传一个本地 fd 数值，而是消息携带的 `GUnixFDList` 中的索引。服务端从列表取得 fd 后读取 `/proc/self/fd/<fd>` 的符号链接；只要链接目标除首字符外还包含 `/`，就把它视为磁盘路径而拒绝。普通重定向文件因此无法通过。

使用 `memfd_create` 创建匿名内存文件，写入精确消息并把偏移复位：

```c
int fd = memfd_create("flag2", MFD_CLOEXEC);
write(fd, "Please give me flag2\n", 21);
lseek(fd, 0, SEEK_SET);

GUnixFDList *list = g_unix_fd_list_new();
int index = g_unix_fd_list_append(list, fd, &error);
GVariant *args = g_variant_new("(h)", index);
```

随后使用 `g_dbus_connection_call_with_unix_fd_list_sync`，将 `list` 与 `(h)` 参数一起调用 `GetFlag2`。服务进程中该 fd 通常显示为 `/memfd:flag2 (deleted)` 一类不含额外路径分隔符的目标，从而通过“不能是磁盘文件”的检查；读取内容严格等于 `Please give me flag2\n` 后返回第二个 flag。

### 第三问：伪造可变的 comm

服务端通过 `org.freedesktop.DBus.GetConnectionUnixProcessID` 得到真实发送进程 PID，但随后只读取 `/proc/<pid>/comm` 并与 `getflag3\n` 比较。`comm` 只是可由进程自身修改的线程名，不是可靠的可执行文件身份。

在发起 `GetFlag3` 前设置名字即可：

```c
#include <sys/prctl.h>

prctl(PR_SET_NAME, "getflag3", 0, 0, 0);
```

然后正常调用无参数的 `GetFlag3` 并解析返回的 `(s)`。也可把提交程序直接命名为 `/dev/shm/getflag3` 后执行，但 `PR_SET_NAME` 更直接地说明检查为何失效。

题目附带的 `getflag3` 样例确实能通过服务端检查，却故意只打印“拿到结果但不展示”。另一条可行路线是向该程序注入负责调用和打印的共享库；不过在当前检查模型下，修改自身 `comm` 已足够，不需要扩大利用链。

### 验证

官方配置只允许 root 拥有该服务名，但默认策略允许其他用户向它发送消息；提交程序以 `nobody` 运行，符合“能调用、不能冒充服务”的边界。三条解法分别命中源码中的字符串比较、`GUnixFDList`/`readlink` 校验和 `/proc/<pid>/comm` 比较，没有依赖临时服务地址。

## 方法总结

- 核心技巧：还原 D-Bus name/path/interface/method；用 `memfd_create + GUnixFDList` 传匿名 fd；用 `PR_SET_NAME` 证明 `comm` 不能承担鉴权。
- 识别信号：D-Bus `h` 类型、`g_dbus_message_get_unix_fd_list`、`GetConnectionUnixProcessID` 和 `/proc/<pid>/comm` 分别指向 FD 传递与调用者身份混淆。
- 复用要点：消息中的 fd 参数是列表索引；传递前必须写入精确字节并复位 offset；安全鉴权应使用 D-Bus policy、Polkit 或不可由调用者随意修改的凭据，而不是进程名。
