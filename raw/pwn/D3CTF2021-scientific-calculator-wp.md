# scientific calculator

## 题目简述

官方总 WP 没有公开完整解法，只说明涉及 CPython 设计层面的安全漏洞，需等待 Python 安全响应团队回应。题目源码显示这是 Python 沙箱/计算器题：用户输入十六进制源码，服务端 `compile(..., "exec")` 后安装 audit hook，只允许 `import` 和 `exec` 两类 audit action，其余 action 触发 `SystemExit`。

源码仓库 README 标明环境是 Debian bullseye 2021/3/7 的 `Python 3.9.1+`，基本等价于修复同类漏洞的 `Python 3.9.2`。由于官方和源码仓库均未公开最终漏洞利用细节，本篇只保留可确认的题目模型和资料边界，不编造 exploit。

## 解题过程

题目源码见 [`zkonge/d3ctf-2021-scientific_calculator`](https://github.com/zkonge/d3ctf-2021-scientific_calculator)。该仓库确认了服务端输入模型、CPython 版本边界和 audit hook 逻辑；由于官方未公开最终漏洞细节，本文只保留可确认部分。

官方总 WP 说明本题解法涉及 CPython 设计层面的安全漏洞；当前公开仓库只保留了题目环境、部署方式和沙箱代码，没有给出可公开的最终 exploit。

源码仓库补充的关键模型如下：

```python
from sys import addaudithook

def hook(action, _):
    if action not in ("import", "exec"):
        raise SystemExit(0)
    raise 0

source = compile(bytes.fromhex(input()), "", "exec")
addaudithook(hook)
del hook, addaudithook
exec(source)
```

也就是说，题目不是普通表达式计算器，而是围绕 CPython audit hook、`compile`/`exec` 和特定 Python 版本行为设计的沙箱逃逸题。源码仓库还说明复现应使用 Debian bullseye 在 2021/3/7 的 `Python 3.9.1+` 包；Docker Hub 的 `Python 3.9.2` 因动态库差异可能无法复现题目环境。

## 方法总结

- 核心技巧：官方未公开最终漏洞；当前只能确认题面是 CPython audit hook 沙箱，输入为十六进制编码的 Python 源码。
- 识别信号：`addaudithook` 白名单极小，只允许 `import`/`exec`，并且题目严格绑定到特定 CPython 3.9.1+/3.9.2 环境。
- 复用要点：遇到 Python 沙箱题要记录解释器精确版本和安全机制；如果官方因安全响应未公开解法，长期 WP 应明确资料边界，避免把猜测写成事实。

