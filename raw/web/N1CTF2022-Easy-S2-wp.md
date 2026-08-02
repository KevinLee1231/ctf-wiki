# N1CTF 2022 - Easy_S2

## 题目简述

题目是一个 Struts2 应用。Java Servlet 的 `security-constraint` 只允许未认证用户访问 `/img/*` 和 `/index.jsp`，其余路径被空的 `auth-constraint` 拒绝；但 Struts2 filter 又接管了 `/*` 的所有请求。目标是利用 Servlet 安全约束和 Struts action 路由解析对象不同这一点，在允许的 URL 前缀下访问受保护的 `flag` action。

本地仓库中的 `easy_s2` 目录只有空占位文件。解法依据比赛队伍保存的 WAR 配置和反编译结果重建，来源为 [N1CTF 2022 Web 复现](https://hujiekang.top/posts/n1ctf-2022/)。关键配置与匹配过程已完整整理如下，不需要依赖外链才能复现。

## 解题过程

### Servlet 层的访问控制

`web.xml` 把所有路由交给 Struts2：

```xml
<filter>
  <filter-name>struts2</filter-name>
  <filter-class>
    org.apache.struts2.dispatcher.filter.StrutsPrepareAndExecuteFilter
  </filter-class>
</filter>
<filter-mapping>
  <filter-name>struts2</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

安全约束先把静态路径列为无需认证：

```xml
<security-constraint>
  <display-name>pass-static</display-name>
  <web-resource-collection>
    <web-resource-name>static</web-resource-name>
    <url-pattern>/img/*</url-pattern>
    <url-pattern>/index.jsp</url-pattern>
  </web-resource-collection>
</security-constraint>
```

另一条约束覆盖 `/*`，并带有空的 `<auth-constraint/>`。它不是“存在某个未给出的角色”，而是没有任何角色获准访问，所以请求其他路径会得到 403。`login-config` 虽然声明了 Basic Auth，但没有角色能通过该约束，直接请求 `/flag.do` 仍不可行。

### Struts2 的 action 配置

`WEB-INF/classes/struts.xml` 把 action 后缀改为 `.do`，并在默认 namespace 中注册 `index` 和 `flag`：

```xml
<struts>
  <constant name="struts.action.extension" value="do"/>
  <package name="struts2" extends="struts-default">
    <action name="index">
      <result>/index.jsp</result>
    </action>
    <action name="flag"
            class="com.mycompany.helloworld.action.FlagAction">
      <result>/WEB-INF/views/jsp/layouts/flag.jsp</result>
    </action>
  </package>
</struts>
```

反编译 `FlagAction` 可确认它读取服务器的 `/flag` 文件并交给 JSP 展示。

### 利用 namespace 回退

Struts2 的 namespace 不是文件系统式的层级目录。按照 [Struts namespace 配置规则](https://struts.apache.org/core-developers/namespace-configuration)，若请求 `/a/b/flag.do` 而 `/a/b` namespace 不存在，框架会回退到默认 namespace，并继续查找名为 `flag` 的 action。

因此下面两个路径最终会命中同一个 action：

```text
/flag.do
/img/flag.do
```

区别在 Servlet 安全层：`/flag.do` 落入受保护的 `/*`，而 `/img/flag.do` 同时匹配明确放行的 `/img/*`。直接访问：

```http
GET /img/flag.do HTTP/1.1
Host: target
Connection: close
```

Servlet 容器允许请求通过，随后 Struts2 回退到默认 namespace，执行 `flag` action 并返回 flag。

另一份比赛 PDF 的一行简解写成 `/image/flag.do`，但保存下来的 `web.xml` 明确放行的是 `/img/*`，因此可复现的正确路径应为 `/img/flag.do`。现有材料没有保留实际 flag 值，这里不凭空补写。

## 方法总结

访问控制绕过常发生在多层路由对同一 URL 的解释不一致时。本题中 Servlet 只按原始路径匹配 `/img/*`，Struts2 却把最后一段解析为 action 并在 namespace 不存在时回退到默认 namespace。审计 Java Web 应用时，应分别记录容器映射、安全约束、filter、框架 namespace 和最终 handler，不能假设它们共享同一套路径语义。
