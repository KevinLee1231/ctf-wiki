---
type: technique
tags: [pwn, sandbox, pyjail, object-graph, capability]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/python-vm-and-proc-sandbox-escape.md
  - ../raw/pwn/pyjails.md
updated: 2026-07-27
---

# Restricted-Runtime Object Graph and Capability Recovery

## 适用场景

Python/JS/模板/解释器沙箱删除危险 builtins、限制字符或只暴露少量对象，但运行时对象图仍能到达类、模块、函数 globals、文件描述符或宿主能力。

## 识别信号

- `eval/exec` 受限、builtins 清空，但对象、异常、函数或类型仍可访问。
- 属性名、括号、引号或字符集受限，而反射/索引/格式化仍可用。
- 沙箱内存在继承 fd、模块缓存、环境变量或宿主回调。

## 最小证据

- 列出入口对象、允许语法/字符和实际过滤顺序。
- 用无副作用表达式证明可遍历到类型、globals 或模块。
- 明确最终能力是文件读、命令、网络还是已继承句柄。

## 解法骨架

1. 从现有对象沿 class/MRO/subclasses/function globals 建图。
2. 用可用语法生成被禁字符串与属性名。
3. 优先恢复最小能力：open、import、已有 module 或 fd。
4. 验证过滤发生在源码、AST、bytecode 还是运行时，必要时切换层级。

## 关键变体

- Python object graph/subclasses。
- JS prototype/constructor/realm escape。
- 模板对象链与宿主 helper。

## 常见陷阱

- 复制依赖固定 `subclasses()` 索引的 payload。
- 只通过本地新版本测试，目标运行时对象顺序不同。
- 获得对象引用后未确认其权限和参数可控性。

## 关联技巧

- [pyjails.md](pyjails.md)
- [python-vm-and-proc-sandbox-escape.md](python-vm-and-proc-sandbox-escape.md)
- [sandbox-capability-and-inherited-channel-bypasses.md](sandbox-capability-and-inherited-channel-bypasses.md)
- [restricted-shell-feature-and-output-channel-escape.md](restricted-shell-feature-and-output-channel-escape.md)

## 原始资料

- [python-vm-and-proc-sandbox-escape.md](../raw/pwn/python-vm-and-proc-sandbox-escape.md)
- [pyjails.md](../raw/pwn/pyjails.md)
