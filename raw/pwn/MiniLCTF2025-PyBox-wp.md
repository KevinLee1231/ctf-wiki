# PyBox

## 题目简述

Flask 的 `/execute` 将表单 `text` 插入 `print("{}")`，先按原始文本检查一组 bad characters，再在 `safe_exec` 内 `decode('unicode_escape')`、AST 检查并放进子进程 `exec`。虽然 HTTP 是入口，最终障碍是 Python jail 逃逸与 SUID `find` 提权：entrypoint 将 flag 设为 root:root、0600，却把 `/usr/bin/find` 设为 4755，因此归 `pwn`。

## 解题过程

### 过滤后解码导致的代码注入

输入验证发生在 Unicode escape 解码之前：

```python
for char in badchars:
    if char in text: return Response("Error", status=400)
code = code.encode().decode('unicode_escape')
output = safe_exec(CODE.format(text))
```

将 `")`、换行、引号等敏感字符写成 `\\x22`、`\\n` 等转义形式，检查时不含 badchars，之后却还原为 `print("")` 外的 Python 语句。该点先证明可注入任意 jail 内代码。

### 绕过审计与 AST 限制

初始安全 builtins 包含可覆写的 `list`、`len` 和 `Exception`。审计函数用它们检查 event/args，故注入代码可先改写 `list=lambda x: True` 与 `len=lambda x: 0`，使 audit hook 不再拒绝后续操作。接着故意抛出异常，从 `Exception` 的 traceback frame 回溯到 `safe_exec` 外层全局，取得真正 `__builtins__`；敏感属性名同样以 Unicode escape 延后还原，避开 AST visitor。

得到 builtins 后，以其 `exec`/`__import__` 执行命令。响应长度被限制为 5，故先把输出写入 `/app/app.py` 等应用用户可写文件，或分片读取，而不是试图直接回显完整 flag。

### SUID 读取最终文件

容器中服务用户无权读 `/m1n1FL@G`，但 `/usr/bin/find` 已被设置 setuid root。取得命令执行后可在授权容器中使用：

```sh
find . -exec cat /m1n1FL@G \; -quit
```

若仍受短回显限制，把输出重定向到可读文件后再以分片方式返回。验证应分开做：先确认注入的受控写入，再确认 find 的有效 uid 读取到了 root-only 文件。未启动题目容器；链路基于 `app.py` 与 `entrypoint.sh`。

## 方法总结

- 核心技巧：利用过滤后 Unicode 解码注入 Python，绕过可重绑定的审计依赖与 AST 名称检查，再用 SUID find 越权读取。
- 识别信号：字符串先过滤后解码、`str.format` 插入代码、异常对象暴露 traceback，以及容器 entrypoint 主动设置 SUID。
- 复用要点：AST 黑名单不是 sandbox；固定解析顺序、不要把不可信代码交给 exec，并移除非必要 SUID 位。
