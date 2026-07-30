# L3akCTF 2024 PySysMagic Writeup

## 题目简述

这是一道 Python 3.10 pyjail。输入同时受到三层限制：

1. 正则拒绝 `()`、引号、数字和普通空格；
2. 随后把当前 `__builtins__.__dict__` 中的所有值都改成 `None`；
3. C 扩展安装全局 audit hook，只允许事件名中首先匹配到 `compile` 或 `exec` 的事件，其他审计事件会直接终止进程。

最终代码通过：

```python
eval(code, {
    "__builtins__": {},
    "_": ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
})
```

执行。唯一主动提供的对象 `_` 是一个类，其名字恰好包含完整大小写字母表。官方解法将字符合成、特殊方法调用、对象子类枚举和底层进程创建组合起来，绕过了三层限制。

## 解题过程

### 1. 在禁用字符下建立基本常量

`False`、`True`、`None` 是关键字常量，不依赖已被清空的 builtins。令：

```text
z = False
o = True
n = z - o
alphabet = _.__name__
```

即可得到 $0$、$1$、$-1$ 和字母表。任意整数可用多个 `o` 相加得到，任意标识符字符串可写成若干 `alphabet[index]` 的连接。普通空格被禁，但制表符没有被过滤，可用制表符分隔推导式中的 token。

### 2. 不写圆括号也能调用函数

Python 的运算符会隐式调用特殊方法。官方 payload 反复改写辅助类 `_` 的方法槽，再用对应语法触发调用：

```text
obj[key]  -> __getitem__
+obj      -> __pos__
obj.attr  -> __getattr__
```

配合列表推导式和赋值表达式，就能完成“取属性、调用无参函数、向列表追加、构造 tuple”等操作。随后从：

```text
obj.__class__.__base__.__subclasses__()
```

按类名寻找 `BuiltinImporter`、`ModuleSpec`、`BaseException`、`tuple` 与 `functools.partial`，避免依赖不稳定的子类索引。

### 3. 修复导入器所需的最小 builtins

清空 builtins 破坏了 `BuiltinImporter.load_module` 的 Python 包装逻辑。`ModuleSpec.__init__.__globals__` 仍引用真实模块全局字典，可由此找回被清空的 builtins 字典。官方脚本再从异常类继承树恢复 `AttributeError`、`KeyError`、`DeprecationWarning`，并只补回加载器所需的少量名称：

```text
getattr, hasattr, tuple, DeprecationWarning,
ImportError, KeyError
```

其中 `getattr`、`hasattr` 和 `tuple` 只需提供满足加载流程的替代 lambda，不必完全还原原始实现。这样 `BuiltinImporter.load_module` 就能加载内建模块 `_posixsubprocess` 与 `os`。

### 4. 绕过 audit hook 执行 readflag

常规的 `os.system` 或 `subprocess.Popen` 会产生被 hook 拦截的审计事件。官方解法直接取：

```text
_posixsubprocess.fork_exec
```

构造其完整位置参数，将可执行文件列表设为 `readflag`。由于不能写圆括号，先用 `functools.partial` 创建对象，再通过 `partial.__setstate__` 写入函数与参数，最后把该对象挂到 `__pos__`，以 `+obj` 触发执行。子进程继承标准输出，因此 `readflag` 读取 `flag.txt` 后直接回显：

```text
L3AK{ok_so_os_wrap_close_works_with_builtin_removal_so_added_audit_sandbox_lol_689db2}
```

官方 `solve.py` 的主要价值是把上述抽象步骤编译成满足过滤器的单行表达式；生成后的 payload 超过 12 KB，且强依赖 Python 3.10 的内部类和 `fork_exec` 参数布局，直接把整串粘入题解反而会掩盖利用原语。

## 方法总结

- 删除 builtins 名称不等于删除运行时能力。已经加载的类、函数的 `__globals__`、异常继承树和内建导入器仍可能保留通往系统功能的引用。
- 禁止圆括号时，应系统检查运算符、订阅、描述符和特殊方法能否形成隐式调用原语。
- Python audit hook 主要覆盖受审计的高层 API；若攻击者能到达不产生同类事件的底层 C 接口，黑名单仍可能失效。
- 这类 payload 与解释器版本高度绑定。复现时应固定题目镜像的 Python 3.10，并按类名搜索对象、核对 `_posixsubprocess.fork_exec` 的实际签名。
