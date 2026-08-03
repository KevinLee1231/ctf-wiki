# easy math 1

## 题目简述

登录题目提供的 SSH 环境后，需要运行 `/home/ctf/easy-math` 并连续回答 10000 道一位十六进制数范围内的乘法题。程序从 `/dev/urandom` 读取一个字节，把低四位和高四位分别作为两个乘数，因此每个乘数都在 $[0,15]$。

真正的限制不在乘法，而在 `check_id()`：

```c
struct stat real, given;
stat("/proc/1/fd/0", &real);
fstat(0, &given);
if (real.st_dev != given.st_dev) return 1;
if (real.st_ino != given.st_ino) return 1;
```

程序要求自己的标准输入与容器中 PID 1 的文件描述符 0 指向同一个内核对象。普通的 `echo ... | /home/ctf/easy-math` 或输入重定向会给子进程换成管道或文件，设备号、inode 不再相同，因而会在考试开始前被拒绝。

## 解题过程

第一问没有隐藏输出：交互式 SSH shell 和程序共享同一个伪终端，直接在该终端中启动程序即可通过文件描述符检查。随后只需在客户端保留同一条交互通道，解析题目并回送乘积。

下面给出核心自动化逻辑。`io` 表示已经申请了 PTY 的 SSH 交互通道；连接参数应替换为题目部署时提供的值，不能使用只执行单条远程命令的非交互接口。

```python
import re

io.send(b"/home/ctf/easy-math\n")
buf = b""

for _ in range(10000):
    while not buf.endswith(b" = "):
        buf += io.recv(1)

    match = re.search(rb"Question \d+: (\d+) \* (\d+) = $", buf)
    if match is None:
        raise RuntimeError(buf)

    a, b = map(int, match.groups())
    io.send(f"{a * b}\n".encode())
    buf = b""

print(io.recvall().decode())
```

若使用 Paramiko，可通过 `client.invoke_shell()` 获得这样的 PTY 通道；若使用 pwntools，则要先进入交互 shell，再发送程序路径。关键点是自动化发生在 SSH 客户端，服务端程序的 fd 0 仍然继承 PID 1 所使用的终端，而不是由本地脚本在服务端另建管道。

全部作答正确后，程序降低到普通用户身份并执行 `cat /home/ctf/flag`，得到：

```text
uiuctf{now do it the fun way :D}
```

这段 flag 同时提示第二问需要把自动求解逻辑放进远端环境，而不能继续依赖可实时读取的终端输出。

## 方法总结

- 核心技巧：在保持 SSH 伪终端不变的前提下，从客户端自动解析并回答 10000 道乘法题。
- 识别信号：程序同时检查 `/proc/1/fd/0` 和自身 fd 0 的 `st_dev`、`st_ino`，说明题目关心的是两个描述符是否引用同一对象，而不只是输入内容。
- 复用要点：遇到 PTY、管道和重定向限制时，应先画清进程与文件描述符继承关系。`isatty()`、路径名或描述符编号相同都不等价于底层对象相同，真正需要比较的是内核对象及其元数据。
