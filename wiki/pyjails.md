---
type: family
tags: [pwn, web, reverse, family, pyjail, sandbox, python, oracle]
skills: [ctf-pwn, ctf-web, ctf-reverse]
raw:
  - ../raw/pwn/pyjails.md
  - ../raw/web/WMCTF2025-guess-wp.md
updated: 2026-07-06
---

# Python Jails

## 作用边界

本页是 Python 受限执行 family，用于从 `eval/exec`、AST 过滤、字符集限制、禁用 builtins、对象链、装饰器链、oracle 型交互和 agent sandbox 中判断逃逸路线。它不是“payload 列表”，而是先确认沙箱模型和最小可用原语。

如果约束已经不是 Python 运行时，而是 bash/rbash、容器边界、Web 模板或原生 pwn，应及时转到相邻页面。

## 识别信号

- 题目暴露 Python 表达式执行、REPL、`eval(payload, globals)`、`exec`、`compile`、AST visitor、blacklist、whitelist、audit hook、agent 工具调用或 Mastermind 式反馈。
- 字符、长度、关键字、括号、引号、点号、下划线、调用、赋值或 import 被限制。
- 输出只有成功/失败、异常类型、长度、位置命中或布尔 oracle。

## 最小证据

- 先枚举语法能力：能否使用字面量、属性访问、下标、调用、lambda、comprehension、walrus、decorator、f-string、异常、编码声明。
- 先枚举命名空间：`globals/locals`、`__builtins__`、`__loader__`、`object.__subclasses__()`、已有模块、已有函数 `__globals__`。
- 对 blacklist/whitelist，记录每个被拦截阶段：输入过滤、AST、字节码、运行时异常、audit hook 还是输出过滤。
- 对 oracle 型 jail，确认反馈可区分性和查询成本，再设计二分/线性恢复。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| `__builtins__` 被删或写错 | 先从对象继承链、函数 `__globals__`、`warnings.catch_warnings`、`__loader__` 找回真实 builtins。 | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| 字符集/关键字/下划线受限 | 先构造字符串、数字和属性访问替代：escape、format、docstring、name mangling、getattr 替代。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| 禁止调用、引号或等号 | 检查 decorator、class body、descriptor、operator overload、f-string key eval、walrus 等隐式执行点。 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |
| AST/compile/audit hook 限制 | 判断过滤发生在 AST 还是 bytecode/runtime；尝试编码声明、`compile` 模式、code object 或版本差异。 | [python-vm-and-proc-sandbox-escape.md](python-vm-and-proc-sandbox-escape.md) |
| Mastermind/布尔 oracle | 先恢复长度，再恢复字符集合和位置；不要追求 shell，先把 secret 当状态机求解。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| agent/tool sandbox | 枚举可调用工具、文件权限、环境变量、异常回显和历史上下文，再决定是 prompt/tool 注入还是 Python 对象链。 | [llm-attacks.md](llm-attacks.md) |
| 其实是 shell/rbash | 切到 shell 页面，不要把系统命令限制误判为 Python jail。 | [bashjails.md](bashjails.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 | 对应路线 |
|---|---|---|
| [WMCTF2025-guess-wp](../raw/web/WMCTF2025-guess-wp.md) | `eval(payload, {'__builtin__': {}})` 在 Python 3 中键名写错，且对象继承链仍可通过 `__subclasses__()` 找到函数 `__globals__['__builtins__']` 取回 `__import__`。 | builtins 恢复 + 对象链逃逸 |
| [D3CTF2021-scientific-calculator-wp](../raw/pwn/D3CTF2021-scientific-calculator-wp.md) | CPython audit hook 沙箱绑定精确版本，先记录允许 action、compile/exec 边界和公开资料缺口。 | 跨页补入 |
| [D3CTF2023-escape-plan-wp](../raw/web/D3CTF2023-escape-plan-wp.md) | Python Web 沙箱过滤数字/字母，先用 Unicode 同形字符、len 和切片构造可执行表达式。 | 跨页补入 |
| [LilacCTF2026-checkin-wp](../raw/web/LilacCTF2026-checkin-wp.md) | Python eval jail 经过 IDNA/正则过滤后仍保留 `vars()`、`dir()` 和可变 `LockedList`；原地 pop/append 修改状态。 | 跨页补入 |
| [VNCTF2026-i-really-really-really-and-revenge-wp](../raw/web/VNCTF2026-i-really-really-really-and-revenge-wp.md) | Pyjail 封数字、字符串和 builtins 后，仍可用布尔造数、类元信息造字符串、`object.__subclasses__()` 找全局模块，并用异常外带。 | 跨页补入 |
| [VNCTF2026-i-really-really-really-ultimate-wp](../raw/web/VNCTF2026-i-really-really-really-ultimate-wp.md) | Unicode 绕过被补后，生成器 `gi_frame.f_back` 仍能回溯栈帧恢复 `builtins`，再拼 `/flag` 并用断言报错泄露。 | 跨页补入 |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖对象链、字符集构造、装饰器、f-string、oracle、audit hook、agent sandbox 和多版本差异，路线分叉明显。
- 不拆成 `subclasses`、decorator、oracle 等小页：当前页面作为 Pwn/Web/Reverse 之间的 jail 二级入口更有查询价值；后续某一路线积累多篇 WP 再拆 technique。
- 不并入 [bashjails.md](bashjails.md)：Python jail 的对象模型、命名空间和 bytecode 语义与 shell jail 不同。

## 常见陷阱

- 直接套 `().__class__.__mro__` payload，没有先枚举 Python 版本和 subclass index。
- 只看 blacklist 字符，没有确认过滤发生在输入、AST、compile 还是 runtime。
- 误把 `__builtins__` 为空当成无解；很多函数对象、loader、warnings 链仍可恢复 builtins。
- Oracle 型题急着 RCE，忽略反馈已经足够恢复 flag。
- 多轮交互时没保存状态，错过 class attribute、全局变量或历史上下文持久化。

## 关联技巧

- [cross-category-triage-family.md](cross-category-triage-family.md)
- [bashjails.md](bashjails.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [python-vm-and-proc-sandbox-escape.md](python-vm-and-proc-sandbox-escape.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [llm-attacks.md](llm-attacks.md)

## 原始资料

- [pyjails.md](../raw/pwn/pyjails.md)
- [WMCTF2025-guess-wp](../raw/web/WMCTF2025-guess-wp.md)
