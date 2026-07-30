# L3akCTF 2024 Fire Checker Writeup

## 题目简述

服务端把真实 flag 作为 `--flag` 参数传给 `chall.py`，再通过 Python Fire 暴露 `check_flag`：

```python
def check_flag(flag, guess, *args, **kwargs):
    if flag == guess:
        return f"Correct! {guess} is the flag!"
    return f"Incorrect, you guessed {guess}, but the flag is {flag}."

fire.Fire(check_flag)
```

外层程序本想过滤所有可打印字符：

```python
BANNED_CHARS = string.printable
args = list(filter(lambda x: x not in BANNED_CHARS, args))
```

但 `x` 是 `input().split()` 后的完整参数。该判断只会删掉“整个参数恰好是一个可打印字符”的项，无法阻止多字符参数。更严重的是，Python Fire 不只调用入口函数，还允许继续访问返回对象的方法；错误信息又直接包含真实 flag。决定性漏洞因此是返回值上的方法链注入，而不是猜测 flag。

## 解题过程

[Python Fire 官方指南](https://google.github.io/python-fire/guide/)说明了两个与本题直接相关的行为：函数返回对象仍可继续作为命令链处理；分隔符会结束当前函数的参数收集，使后续 token 作用于返回值。设置自定义分隔符后，可以先让 `check_flag` 返回泄露 flag 的错误字符串，再连续调用字符串的 `replace` 方法，把它改造成外层程序期待的成功文本。

官方 payload 为：

```text
AA XX replace "Incorrect," "Correct!" +1 replace "you\x20guessed\x20AA,\x20but\x20the\x20flag\x20is\x20" "" +1 replace "." "\x20is\x20the\x20flag!" +1 replace "\n" "" +1 -- --separator XX
```

各部分的作用如下：

1. `AA` 作为错误猜测，触发包含真实 flag 的返回值；
2. `XX` 是通过 `-- --separator XX` 设置的 Fire 分隔符，后面的 `replace` 开始处理返回字符串；
3. 每个 `+1` 被解析为 `str.replace(old, new, count)` 的 `count=1`，同时明确填满第三个可选参数，使下一项 `replace` 继续成为方法名；
4. `\x20` 在 Fire 解析字符串字面量后成为空格，避免输入先被 `split()` 拆散；
5. 最后一组替换移除序列化结果末尾的换行。

设真实 flag 为 `F`，四次替换的文本变化是：

```text
Incorrect, you guessed AA, but the flag is F.
Correct! you guessed AA, but the flag is F.
Correct! F.
Correct! F is the flag!
```

最终输出恰好等于外层检查的 `f"Correct! {FLAG} is the flag!"`，于是服务端主动打印：

```text
L3AK{tR4NSF0RMS_iNBuiL7_1n_CLi5_WHO_KneW?!}
```

## 方法总结

- “把函数变成 CLI”的框架可能暴露返回对象的属性和方法；安全边界不能只审计入口函数参数。
- 黑名单逐 token 比较与逐字符检查不是一回事。本题的过滤器只删除单字符参数，任何多字符 payload 都可穿过。
- 即使无法直接读取变量，只要错误消息包含秘密且最终判断只比较字符串，就可能通过返回值变换伪造成功路径。修复时应避免把 flag 放入子进程参数和错误文本，并限制 CLI 框架可达的对象表面。
