# HackINI2024 Fort De Vaux

## 题目简述

题目是一个登录页面。服务端只接受硬编码的学生账号，并在 JSON 响应中返回角色 `student`；前端混淆 JavaScript 却根据响应里的角色决定是否展示内置 flag。目标是修改服务端到浏览器之间的响应，让客户端误以为当前用户是管理员。

## 解题过程

源码给出的有效凭据为：

```text
username: student
password: student-2023
```

正常登录响应是：

```json
{"message":"Login successful","role":"student"}
```

服务端没有管理员登录分支，也没有在后端基于管理员权限返回 flag。真正的错误是前端把响应中的 `role` 当成可信授权结论，并在混淆脚本中保留了敏感结果。

使用拦截代理提交登录请求，在响应到达浏览器前把角色改成：

```json
{"message":"Login successful","role":"admin"}
```

浏览器随后进入管理员显示分支，解出并呈现：

```text
shellmates{ch4n1nG_re$p0NsE_V$_cLIenT_$1d3_LogIc}
```

这个操作只修改客户端收到的副本，不会让服务器端会话真正升级；能够得到 flag 恰好说明敏感数据和授权逻辑错误地放在了客户端。

## 方法总结

前端响应、DOM 状态和 JavaScript 变量都由用户完全控制，不能承担访问控制。代码混淆也只是增加阅读成本，无法保护嵌入其中的 flag。正确设计应让服务器根据不可伪造的会话状态执行管理员检查，并且只在检查通过后由后端返回敏感数据。
