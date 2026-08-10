# 蛛蛛...嘿嘿♥我的蛛蛛

## 题目简述

页面中只有一个指向下一页的“点我试试”链接，后续页面继续给出相同形式的链接。需要让爬虫沿链接链不断访问，直到页面不再出现下一跳；最终 flag 不在 HTML 正文，而在终点响应头中。

## 解题过程

先请求起始页面，提取形如下面的链接：

```html
<a href="下一页地址">点我试试</a>
```

每次访问后继续解析新的 `href`。使用 `urljoin` 处理相对路径，并复用一个 `Session`，可以避免手工拼接 URL 以及遗漏会话状态：

```python
import re
from urllib.parse import urljoin

import requests

session = requests.Session()
url = "https://challenge.example/start"

while True:
    response = session.get(url, timeout=10)
    response.raise_for_status()

    links = re.findall(
        r'<a\s+href="([^"]+)">点我试试</a>',
        response.text,
    )
    if not links:
        print(dict(response.headers))
        break

    url = urljoin(url, links[0])
```

当正文不再包含目标链接时，检查最后一次响应的全部响应头，而不是只打印页面文本。题目把结果放在自定义的 `fi4g` 响应头中。

这道题公开实例记录中的具体 flag 并不完全一致，说明值会随实例或访问链变化。例如一份参赛者记录给出的终点响应头是：

```text
fi4g: hgame{202418360e93093582ff7358f3b3829d3f733935bef5686eeb568e9848b779c1}
```

该样例及逐跳抓取过程可见 [YuGao 的 HGAME 2022 Week1 记录](https://sxyugao.top/p/d379320f)。实际作答应以自己实例最后一次响应中的 `fi4g` 值为准，不能把上述动态样例当作所有环境通用的固定 flag。

## 方法总结

本题考查最基础的定向爬取和 HTTP 响应检查。循环终止条件应由“下一跳链接是否存在”决定，并在终点同时检查正文与响应头。使用 `Session` 保留 Cookie、使用 `urljoin` 解析相对地址、为请求设置超时和状态码检查，能让脚本比简单字符串拼接更稳定。
