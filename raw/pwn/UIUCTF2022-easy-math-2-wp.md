# easy math 2

## 题目简述

第二问仍然运行同一个 `easy-math` 二进制，但外层启动脚本把整个 nsjail 的 stdout 和 stderr 重定向到临时文件，只有 shell 退出后才统一显示：

```bash
kctf_drop_privs kctf_pow nsjail --config /home/user/nsjail2.cfg \
    > "$TEMP_OUTPUT" 2>&1 || :
cat "$TEMP_OUTPUT"
```

因此，客户端无法像第一问那样边读题目边发答案。与此同时，程序仍要求自身 fd 0 与 `/proc/1/fd/0` 指向同一个对象。解法必须在远端 shell 内启动一个代理进程，由它实时读取 `easy-math` 的输出、计算答案并写回输入。

## 解题过程

### 构造满足检查的管道拓扑

先创建管道 `(r, w)`，然后把代理 Python 进程的 fd 0 替换为读端 `r`。再让 `easy-math` 的标准输入也来自同一个读端：

```text
答案生成器 ──write──> w
                      │
                      └──pipe──> r <── Python(PID 1) 的 fd 0
                                      └── easy-math 的 fd 0
```

`easy-math` 比较的是设备号与 inode，所以两个描述符可以是不同编号，只要它们引用同一个管道读端就会通过。外层 shell 必须用 `exec python3 ...` 替换自身；这样 Python 保留容器内 PID 1，`/proc/1/fd/0` 才会指向刚设置的管道。若只是普通地启动 Python 子进程，PID 1 仍是原 shell，检查仍会失败。

### 在远端进程内部实时求解

仓库给出的官方脚本可整理为：

```python
import os
import subprocess

r, w = os.pipe()

# dup 会依次占用刚关闭的 0 和 1；PID 1 的 fd 0 因而指向 r。
os.close(0)
os.close(1)
os.dup(r)
os.dup(r)

proc = subprocess.Popen(
    "/home/ctf/easy-math",
    stdin=r,
    stdout=subprocess.PIPE,
)

for _ in range(10000):
    question = b""
    while not question.endswith(b" = "):
        byte = proc.stdout.read(1)
        if not byte:
            raise RuntimeError("easy-math exited early")
        question += byte

    fields = question.split()
    answer = int(fields[2]) * int(fields[4])
    os.write(w, f"{answer}\n".encode())

tail = proc.communicate()[0]
os.write(2, tail)
```

将代码保存为 `/tmp/solve.py` 后，在远端 shell 中执行：

```bash
exec python3 /tmp/solve.py
```

外层服务虽然暂不把输出发回客户端，但 `proc.stdout` 是代理进程自己创建的内部管道，Python 仍能逐字节读取每一道题。完成 10000 次交互并退出后，临时文件才被外层脚本打印，最终可见：

```text
uiuctf{excellent execution}
```

## 方法总结

- 核心技巧：用 `exec` 让求解器成为 PID 1，再让 PID 1 和目标程序的 fd 0 共同引用一个管道读端；求解器从目标 stdout 实时取题并向管道写答案。
- 识别信号：服务端延迟所有网络输出，但允许在 shell 内运行进程；这意味着交互闭环应移到服务端内部，而不是继续尝试客户端同步读写。
- 复用要点：分析 `/proc/<pid>/fd/<n>` 检查时，要同时跟踪 PID namespace、`exec` 是否保持 PID、`dup` 的最低空闲描述符规则，以及 `subprocess` 为 stdin/stdout 建立的实际管道方向。
