# easy math 3

## 题目简述

第三问修补了第二问的一条非预期解法。外层脚本不再直接运行 nsjail，而是通过 `script` 分配一层新的伪终端；输入期间连本地回显都不可见，记录内容仍要等 shell 退出后才返回：

```bash
script -c "kctf_drop_privs kctf_pow nsjail --config /home/user/nsjail3.cfg" \
    -q "$TEMP_OUTPUT" >/dev/null 2>/dev/null
```

题目明确说明，如果第二问使用的是预期解法，则脚本可以原样工作。预期解法的关键从来不是外层终端，而是容器内部 PID 1 与 `easy-math` 共享同一管道读端，因此新增的 PTY 包装不会破坏该关系。

## 解题过程

在没有回显的 SSH 会话中输入以下操作：先把求解器写到 `/tmp/solve.py`，再用 `exec` 启动它。核心代码如下：

```python
import os
import subprocess

r, w = os.pipe()

os.close(0)
os.close(1)
os.dup(r)  # 新 fd 0：管道读端
os.dup(r)  # 新 fd 1：同一读端；后续可回传内容改写到 fd 2

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

os.write(2, proc.communicate()[0])
```

启动时必须执行：

```bash
exec python3 /tmp/solve.py
```

进程关系可概括为：

1. nsjail 内的登录 shell 原本是 PID 1；
2. `exec` 只替换进程映像，不改变 PID，因此 Python 成为新的 PID 1；
3. Python 把自己的 fd 0 换成管道读端；
4. `easy-math` 的 fd 0 也由同一读端复制而来，`st_dev` 和 `st_ino` 检查通过；
5. Python 直接读取子进程 stdout 并向管道写入答案，整个闭环都在容器内部完成；
6. 求解器退出后，外层 `script` 才把录制结果交给客户端。

新增的伪终端只改变了 nsjail 外层的输入输出传递方式，并没有改变第 2 至第 5 步的内部文件描述符拓扑。最终得到：

```text
uiuctf{file descriptors are literally magic}
```

## 方法总结

- 核心技巧：复用第二问的 `exec + pipe` 文件描述符布局，在完全盲交互的外层仍由容器内代理进程完成实时问答。
- 识别信号：补丁只增加 PTY 和输出录制层，而目标检查仍比较 `/proc/1/fd/0` 与自身 fd 0；决定性原语没有被修复。
- 复用要点：面对补丁题，应比较补丁前后的安全不变量。终端是否回显、输出何时传给网络，与两个进程是否引用同一管道是不同层面的问题；只要后者不变，原利用链就可以继续成立。
