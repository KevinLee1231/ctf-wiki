# ez_ssti

## 题目简述

Flask 应用在启动时读取环境变量中的 flag，随后调用 `os.unsetenv("flag")` 删除环境变量，但模块级变量 `flag` 仍保留原值。404 处理函数把用户可控的请求 URL 先拼入模板字符串，再交给 Jinja2 渲染：

```python
flag = os.getenv("flag")
os.unsetenv("flag")

@app.errorhandler(404)
def page_not_found(e):
    return render_template_string(
        "<h1>The Url {} You Requested Can Not Found</h1>".format(request.url)
    )
```

因此注入点不在正常首页，而在任意不存在的路径中。Jinja2 没有设置过滤或沙箱，目标是从模板对象访问 Python 全局对象，最终读取 `__main__` 模块中的 `flag` 变量。

## 解题过程

先访问包含 `{{7*7}}` 的不存在路径。响应中出现 `49`，证明 URL 内容经过了第二次 Jinja2 模板解析，而不是仅作为普通字符串显示。

常见的 SSTI 命令执行 payload 可以通过 Python 内置函数导入 `os`，但本题执行 `env` 已经读不到被删除的环境变量。关键区别是：`unsetenv()` 只影响进程环境，不会清空已经赋值的 Python 全局变量。

运行脚本的模块名为 `__main__`，而 `sys.modules` 保存了当前进程已加载的模块对象。通过未定义变量 `x` 对应的 Jinja `Undefined` 对象进入其 `__init__.__globals__`，取得 `__builtins__` 和 `__import__`，再导入 `sys`：

```jinja2
{{x.__init__.__globals__['__builtins__']['__import__']('sys').modules['__main__'].flag}}
```

将该表达式放入 404 路径后，模板会直接输出模块级变量：

```text
0xGame{Do_You_Want_To_Be_A_SSTI_Master?}
```

## 方法总结

- 核心技巧：利用 404 页面中的无过滤 Jinja2 SSTI，并从 `sys.modules['__main__']` 读取已缓存的模块全局变量。
- 识别信号：`render_template_string()` 的参数由请求数据拼接而成时，应检查是否存在二次模板解释；环境变量被删除后，还要检查值是否已复制到其他对象。
- 复用要点：不要把“环境变量不存在”等同于“秘密已从进程内存消失”；SSTI 的目标可以是全局变量、配置对象或已加载模块，不一定要执行 shell。
