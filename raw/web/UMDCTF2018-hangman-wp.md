# UMDCTF 2018 - Hangman

## 题目简述

这是一个 Flask 猜词游戏。玩家提交的“字母”会被追加到数据库字段 `tried`，而计分时服务端又把整个字段当作 Jinja 模板渲染，形成服务端模板注入。

## 解题过程

漏洞位于 `Game.errors`：

```python
env = Environment()

@property
def errors(self):
    return env.from_string(
        self.tried,
        globals=globals()
    ).render()
```

`try_letter` 没有限制输入只能是单个字母：

```python
self.tried += " " + letter
```

因此新建一局后，在 `letter` 参数中提交：

```jinja2
{{ app.flag }}
```

模板渲染时显式传入了模块的 `globals()`，其中包含全局 Flask 对象 `app`。访问或触发返回 `game.errors` 的请求后，表达式会读取：

```python
app.flag = "UMDCTF-{hows-it-hanging}"
```

得到 flag：

```text
UMDCTF-{hows-it-hanging}
```

需要注意，当前源码中该字符串的 SHA-256（无论是否附加换行）都不等于 README 保存的 `7ddbce...`。公开仓库的源码与摘要不是同一版本；这里保留可由当前服务端代码直接返回的结果，并明确记录这一差异。

## 方法总结

模板注入的根因不是 Flask 页面本身，而是把持久化的用户输入再次交给模板解释器。审计时要追踪数据从表单、数据库到模板的完整路径；若源码与摘要冲突，应如实披露，不能编造一个无法从现有工件推出的字符串。
