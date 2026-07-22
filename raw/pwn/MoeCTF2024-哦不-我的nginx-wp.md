# 哦不！我的 nginx！

## 题目简述

目标系统中的 glibc、动态加载器和常用命令已损坏，只有已经运行的 Bash 还能工作；需要在受限 Shell 中恢复最小用户态工具链和动态链接环境，最终启动 nginx。决定性原语是 Bash 内建命令、重定向和 `/dev/tcp`，因此按执行环境逃逸归入 Pwn，而不是云控制面类题目。

## 解题过程

### 1. 用 Bash 内建能力恢复静态 BusyBox

`printf`、`read`、`exec`、重定向和 `/dev/tcp` 都由 Bash 自身实现，不需要启动外部程序。直接把任意二进制放进 Bash 变量会丢失 NUL 字节，所以先在题目提供的完整 `neighbor` 主机上把 BusyBox 转为 ASCII 的 `\xNN` 转义流：

```bash
# neighbor：busybox 可从下文保留的官方下载地址取得。
od -An -v -t x1 ./busybox \
  | awk '{for (i=1; i<=NF; i++) printf "\\x%s", $i}' \
  | nc -lvp 8000
```

在损坏主机上只用 Bash 内建命令接收。每次读取 60000 个字符，恰好是四的倍数，不会从中间切断 `\xNN`；`printf '%b'` 再把转义流还原成原始字节：

```bash
exec 7<>/dev/tcp/127.0.0.1/8000
: > /bin/cp
while IFS= read -r -N 60000 chunk <&7; do
    printf '%b' "$chunk" >> /bin/cp
done
printf '%b' "$chunk" >> /bin/cp
exec 7>&-
```

这里必须选择一个原本就存在且带执行位的损坏文件（示例为 `/bin/cp`）作为覆盖目标；重定向截断已有 inode 时会保留权限位。BusyBox 会根据 `argv[0]` 选择同名 applet，所以写入后的 `/bin/cp` 可以直接作为 `cp` 使用，再复制出其他命令：

```bash
/bin/cp /bin/cp /bin/chmod
/bin/cp /bin/cp /bin/cat
/bin/cp /bin/cp /bin/nc
```

可复现所需的上游资源是 [BusyBox 官方静态二进制目录](https://busybox.net/downloads/binaries/)，其中包含题解使用的 `1.35.0-x86_64-linux-musl` 构建。静态 musl 版本不依赖目标机已经损坏的 glibc/ld。

另一条可选路线是 Bash 的 `enable -f`：共享对象只需可读即可由当前 Bash 进程加载，不要求文件本身有执行位。[bash-loadables 项目](https://github.com/NobodyXu/bash-loadables) 提供了 `enable -f /path/to/loadable builtin_name` 的加载方式和文件操作类内建扩展示例，可借此恢复 `chmod`；这不是主解所必需的依赖。

### 2. 恢复动态加载环境

有了 BusyBox 的 `nc` 后，不再需要十六进制转义。分别从 `neighbor` 发送与 nginx 环境匹配的 libc 和动态加载器：

```bash
# neighbor
cat /usr/lib/x86_64-linux-gnu/libc.so.6 | nc -lnvp 8000
cat /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 | nc -lnvp 8001
```

```bash
# broken
nc 127.0.0.1 8000 > /usr/lib/x86_64-linux-gnu/libc.so.6
nc 127.0.0.1 8001 > /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
chmod +x /usr/lib/x86_64-linux-gnu/libc.so.6
chmod +x /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

两份文件必须来自兼容的架构和发行版；只恢复 libc 而不恢复 ELF 解释器，动态程序仍无法启动。

### 3. 启动检查服务

```bash
nginx -g 'daemon off;'
```

等待检查请求到达后结束前台 nginx，再读取 `/var/log/nginx/access.log`，flag 会出现在访问记录中。

## 方法总结

题目的核心不是 nginx 配置，而是从“仅存活一个 Bash 进程”的环境逐级自举：先用纯内建能力传入无动态依赖的 BusyBox，再用 BusyBox 恢复文件传输和权限工具，最后补齐成对的 libc 与动态加载器。传二进制时先转 `\xNN` 是为了绕过 Bash 变量不能保存 NUL 字节的限制。
