# AFL sandbox

## 题目简述

服务端接收十六进制 shellcode，将其保存为 `/tmp/shellcode.bin`，再用 AFL 对加载该 shellcode 的 `harness` 进行 5 分钟模糊测试。目标是在标准输出和标准错误被 AFL 重定向、系统调用又受 seccomp 限制的情况下读取 flag 并把内容带回客户端。

题目使用的是经典 [American Fuzzy Lop（AFL）](https://github.com/google/AFL/tree/master)。AFL 通过共享内存收集覆盖率，并通过文件描述符 198、199 与目标内的 forkserver 通信；这些本应只属于模糊测试框架的通道，正是预期解的逃逸面。

## 解题过程

### 审计执行边界

`harness.c` 将 shellcode 只读、可执行地映射到 `0x10000`，并在 `0x20000` 映射一页可读写匿名内存，然后直接调用入口。seccomp 仅允许：

```text
open, read, write, exit, exit_group, brk, shmat
```

因此不能直接建立网络连接，也不能调用 `execve`。此外，AFL 启动目标时会重定向 stdin，并关闭或重定向目标的 stdout、stderr，所以即使 `open`、`read` 成功，直接 `write(1, flag, n)` 也回不到客户端。

另一方面，`shmat` 在常规 shellcode 题中很少出现。结合环境变量字符串 `__AFL_SHM_ID` 和 AFL 固定使用的 198、199 号描述符，可以判断题目希望 shellcode伪装成 forkserver。

### 预期思路：伪造 AFL forkserver

shellcode 可从 `envp` 中找到 `__AFL_SHM_ID=<id>`，把十进制 id 转为整数，再调用：

```c
uint8_t *trace_bits = shmat(shm_id, NULL, 0);
```

AFL 旧版 forkserver 的最小协议为：

1. 目标向 fd 199 写入 4 字节握手值，表示 forkserver 已启动；
2. 每轮从 fd 198 读取 4 字节控制值；
3. 向 fd 199 写入 4 字节“子进程 PID”；
4. 处理当前 fd 0 中的测试用例并修改共享覆盖图；
5. 向 fd 199 写入 4 字节等待状态，例如 `0` 表示正常退出；
6. 回到第 2 步。

题目不要求真的 `fork`，只要按协议回包，AFL 父进程就会相信一次执行已经结束。shellcode 可以读取 `/home/ctf/flag`，再把字符或正确前缀编码进 `trace_bits`；AFL 会把这些变化当作新覆盖率，并在父进程可见的覆盖统计、队列或崩溃状态中体现。也可以根据 flag 位伪造崩溃状态，形成 crash 侧信道。仓库中的 `example.c` 已给出环境变量解析、`shmat`、握手和循环骨架，真正需要替换的是覆盖率编码逻辑。

### 非预期解：写入 `random.py`

源码还存在更直接的跨连接文件投毒。`xinetd` 以 root 启动 `/home/ctf/wrapper.py`，脚本开头执行：

```python
import os
import random
```

Python 会优先从脚本所在目录搜索同名模块。当前目标进程虽然没有可用 stdout，但 seccomp 允许 `open` 和 `write`，且进程拥有 `/home/ctf` 的写权限。因此第一次连接上传的 shellcode 可以创建 `/home/ctf/random.py`，内容为：

```python
print(open("/home/ctf/flag", "r").read(), flush=True)
raise SystemExit
```

shellcode 的工作仅是执行等价于下面的系统调用序列：

```text
fd = open("/home/ctf/random.py", O_WRONLY | O_CREAT | O_TRUNC, 0644)
write(fd, python_source, len(python_source))
exit(0)
```

这里不需要 `close`：进程退出时内核会关闭描述符。第一次连接结束后重新连接，新的 root 权限 `wrapper.py` 在执行工作量证明之前导入本地 `random.py`，投毒模块便通过正常的网络 stdout 打印 flag 并退出。这个解法绕过的是 Python 模块搜索路径和服务权限配置，而不是 seccomp 本身。

## 方法总结

预期解利用 AFL 自身的控制面：`__AFL_SHM_ID` 暴露共享覆盖图，fd 198/199 暴露 forkserver 协议，二者组合后可把无法直接输出的数据编码给父进程。非预期解则说明沙箱安全不能只看允许的系统调用；只要受限代码仍能写入高权限解释器的模块搜索目录，下一次启动就会把普通文件写入升级为 root 代码执行。修复时应让服务降权运行、将工作目录设为不可写，并使用隔离的 Python 模块路径或 `-I` 隔离模式。
