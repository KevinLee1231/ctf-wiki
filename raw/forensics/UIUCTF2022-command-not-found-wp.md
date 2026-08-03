# Command Not Found

## 题目简述

远程环境启动一台以 Btrfs 为根文件系统的 Linux 虚拟机，并在交出 shell 前自动执行一组命令：创建 FIFO，让一个 `cp` 进程永久阻塞；显示进程列表和 `btrfs-find-root /dev/vda` 的结果；最后执行 `rm -rfv / --no-preserve-root`。此时根目录中的普通文件几乎全部被删除，但内核、Bash、已运行进程、块设备和网络连接仍然存在。

题面特别说明 flag 不在磁盘上。源码也印证了这一点：真正的 `/usr/local/bin/get_flag` 只是从 I/O 端口 `0x2022` 逐字节读取，而 QEMU 补丁才把 flag 保存在该虚拟设备中。因此目标是从删除后的系统恢复 `get_flag` 可执行文件，再运行它访问端口设备；直接 carving flag 字符串没有意义。

## 解题过程

### 保留两个删除前的锚点

自动命令的输出中有两项必须先记下：

1. `cp this_copy_will_never_finish right? &` 后显示的后台进程 PID，记为 `CP_PID`；
2. `btrfs-find-root /dev/vda` 输出的 `Found tree root at ...` 数值，记为 `TREE_ROOT`。

FIFO 没有写端，所以 `cp` 一直阻塞。即使其磁盘目录项随后被删除，正在运行的程序映像仍能通过 `/proc/CP_PID/exe` 访问。Btrfs 的旧树根则允许 `btrfs restore -t` 绕过当前已删除的根树，从旧元数据中恢复目标路径。

### 用存活的 `cp` 制造可执行落点

删除完成后进入仍挂载于 tmpfs 的 `/tmp`，用存活的 `cp` 复制它自己的程序映像：

```bash
cd /tmp
/proc/CP_PID/exe /proc/CP_PID/exe busybox
```

此时名为 `busybox` 的文件其实还是 `cp`，但它已经带有可执行权限。这个权限很重要：稍后只用 Bash 重定向替换文件内容时，已有的执行位不会消失。

磁盘上的 `cat` 也已被删掉，不过当前 shell 是 Bash，可以仅靠内建的 `read` 与 `printf` 定义一个二进制安全的替代函数。官方 healthcheck 使用的实现如下：

```bash
cat() {
  cat_inner() {
    while LANG=C IFS= read -r -d '' DATA || (printf '%s' "$DATA"; false); do
      printf '%s\0' "$DATA"
    done
  }

  if [[ $# -eq 0 ]]; then
    cat_inner
  else
    for FILE in "$@"; do
      if [[ $FILE == '-' ]]; then
        cat_inner
      else
        cat_inner < "$FILE"
      fi
    done
  fi
}
```

循环以 NUL 为分隔符读取；每次把被 `read -d ''` 去掉的 NUL 补回，因此可以转存包含零字节的 ELF 文件。

### 通过 Bash 的 `/dev/tcp` 手工下载工具

QEMU 用户网络的宿主侧 `10.0.2.2:2121` 运行匿名 FTP，并提供静态 `busybox` 与 `btrfs`。无需外部客户端，Bash 自己即可打开控制连接：

```bash
exec 3<>/dev/tcp/10.0.2.2/2121
read LINE <&3; printf '%s\n' "$LINE"
printf 'USER anonymous\r\n' >&3
read LINE <&3; printf '%s\n' "$LINE"
printf 'PASS\r\n' >&3
read LINE <&3; printf '%s\n' "$LINE"
printf 'EPSV\r\n' >&3
read LINE <&3; printf '%s\n' "$LINE"
```

`EPSV` 响应形如 `229 Entering Extended Passive Mode (|||PORT|)`。把其中的 `PORT` 代入数据连接，然后请求二进制传输：

```bash
exec 4<>/dev/tcp/10.0.2.2/PORT
printf 'TYPE I\r\n' >&3
read LINE <&3; printf '%s\n' "$LINE"
printf 'RETR busybox\r\n' >&3
read LINE <&3; printf '%s\n' "$LINE"
cat <&4 > busybox
```

重定向会截断原来的 `cp` 内容并写入真正的 BusyBox，同时保留原文件的执行权限。现在可以用它下载和授权 Btrfs 恢复工具：

```bash
./busybox wget ftp://10.0.2.2:2121/btrfs
./busybox chmod a+x ./btrfs
```

### 从旧 Btrfs 树恢复 `get_flag`

根目录被删后原 `/dev/vda` 节点也不存在，但内核中的 virtio 块设备仍在，题目构建固定使用主设备号 254、次设备号 0。重新创建设备节点并只恢复需要的目录：

```bash
./busybox mknod vda b 254 0
./busybox mkdir -p restore
./btrfs restore -ivt TREE_ROOT \
  --path-regex '^/(|usr(|/local(|/bin(|/.*))))$' \
  vda restore/
./busybox chmod a+x restore/usr/local/bin/get_flag
restore/usr/local/bin/get_flag
```

恢复出来的程序调用 `ioperm(0x2022, 1, 1)`，再循环执行 `inb(0x2022)`，所以最终 flag 来自 QEMU 的端口 I/O 设备而不是磁盘。仓库元数据记录的结果为：

```text
uiuctf{what_could_possibly_go_wrong_be6e0fbe}
```

仓库中的 `healthcheck.py` 在完成 FTP 的 `RETR busybox` 协商后存在无条件 `exit(0)`，故后半段恢复命令在该脚本中实际不可达；上面的完整链条依据同一文件中保留的后续代码、构建脚本提供的 FTP 文件以及 QEMU 设备补丁归纳，不能把现有 healthcheck 的成功误写成端到端验证。

## 方法总结

- 核心技巧：利用 `/proc/<pid>/exe` 找回仍在运行但目录项已删除的程序，以 Bash 内建能力重建下载通道，再借助旧 Btrfs 树根恢复指定文件。
- 识别信号：删除发生在运行中的系统、题面刻意保留阻塞进程并打印 `btrfs-find-root`、宿主侧又暴露恢复工具，这些都是“活体文件系统恢复”而非普通磁盘 carving 的提示。
- 复用要点：先保存 PID 与树根地址；二进制传输必须补回 NUL；`btrfs restore` 应限定路径，避免恢复整个系统。还要区分“恢复取 flag 的程序”和“从磁盘恢复 flag”——本题只有前者成立。
