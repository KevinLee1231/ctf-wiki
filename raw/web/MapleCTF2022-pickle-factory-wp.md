# pickle-factory

## 题目简述

服务只接受 JSON，再由服务器调用 `pickle.dumps()`，所以客户端不能直接上传任意 pickle。问题在查看页面：程序先把保存的 pickle 字节对象转成 `str(...)`，拼进模板源码，然后才调用普通 Jinja2 `Environment.from_string()`。JSON 字符串中的 Jinja 表达式因此会在查看时被当成模板执行。

## 解题过程

先创建包含探测表达式的 JSON：

```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

虽然它位于 pickle 的 bytes repr 中，`{{...}}` 仍进入最终模板语法。查看返回列表并记录三个运行时类型的索引：`_pickle.Unpickler`、`tempfile.SpooledTemporaryFile` 和 `bytes`。索引随 Python 构建和导入顺序变化，不能硬编码。

本地定义带 `__reduce__` 的对象，使 `pickle.dumps()` 生成调用 `os.system(command)` 的恶意 pickle；把所得每个字节转成整数数组。第二个 Jinja 载荷通过刚找到的类完成等价操作：

```python
data = bytes([128, 4, ...])
f = SpooledTemporaryFile()
f.write(data)
f.seek(0)
Unpickler(f).load()
```

Jinja 中分别用 `{% set %}` 保存类和对象，再用 `{{ f.write(d) }}`、`{{ f.seek(0) }}` 和 `{{ u.load() }}` 触发方法。命令读取 `flag.log` 或外带其内容，得到：

```text
maple{he_was_fired_and_so_was_she}
```

## 方法总结

“由服务器生成 pickle”并不能抵消后续模板注入；真正的漏洞边界是把数据的字符串表示拼到模板源码中。应使用固定模板并把 pickle repr 作为变量传入，同时使用 Jinja sandbox 和最小权限进程。利用端通过运行时枚举类型比固定 `__subclasses__()` 下标更稳健。
