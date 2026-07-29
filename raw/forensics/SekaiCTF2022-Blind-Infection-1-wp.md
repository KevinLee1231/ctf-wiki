# Blind Infection 1

## 题目简述

附件是一次经过裁剪的 Ubuntu 用户目录快照，`Documents` 与 `Pictures` 中的文件均已被加密。题面提示用户曾把部分文档备份到“链接”，第一问的目标是从系统证据中找回这些在线备份。

决定性证据不在空白的 `.bash_history`，而在 Firefox 的访问历史数据库。检索其中的 Paste 站点链接，再批量检查响应内容即可找到 flag。

## 解题过程

先检查 `etc/passwd`，确认主要用户为 `sekaictf`：

```text
root:x:0:0:root:/root:/bin/bash
sekaictf:x:1000:1000:sekaictf,,,:/home/sekaictf:/bin/bash
```

Ubuntu 22.04 默认以 Snap 形式安装 Firefox。本题的用户配置位于：

```text
home/sekaictf/snap/firefox/common/.mozilla/firefox/
└── p3zapakd.default/
    └── places.sqlite
```

Firefox 把访问过的 URL 保存在 `moz_places` 表。题面说备份是“链接”，而历史记录中又反复出现 `paste.c-net.org`，因此直接执行：

```sql
SELECT url
FROM moz_places
WHERE url LIKE '%paste%';
```

查询得到 49 个候选粘贴链接。无需在图形界面逐条打开，可以直接用 SQLite 和 HTTP 请求自动筛选：

```python
import sqlite3
import requests

db = sqlite3.connect("places.sqlite")
urls = [
    row[0]
    for row in db.execute(
        "SELECT url FROM moz_places "
        "WHERE url LIKE '%paste%'"
    )
]

for url in urls:
    body = requests.get(url, timeout=10).text
    if "SEKAI{" in body:
        print(body)
```

用户的浏览行为具有固定模式：先搜索某个主题，再访问若干结果，最后把相应文档内容保存到一个随机双词路径的粘贴页面。文件名与粘贴内容主题对应，因此这些 URL 确实是加密文档的在线备份，而不是无关浏览记录。

筛选结果为：

```text
SEKAI{R3m3b3r_k1Dz_@lway5_84cKUp}
```

## 方法总结

浏览器历史是用户行为取证的重要证据源。遇到 Linux 桌面快照时，不能只查传统的 `~/.mozilla`，还要考虑 Snap、Flatpak 等沙箱化安装路径。面对大量历史记录，应先用 SQL 按域名和关键词收窄范围，再用脚本做内容匹配，同时保留“为什么这些链接与目标文件有关”的证据链。
