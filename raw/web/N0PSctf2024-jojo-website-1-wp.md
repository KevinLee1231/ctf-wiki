# Jojo Website 1

## 题目简述

网站允许登录用户修改自己的用户名和密码。资料页把数据库用户 ID 放在隐藏表单字段中，但服务端直接信任客户端提交的 `id`，没有确认它是否等于当前会话中的 `$_SESSION['id']`。

用户列表公开显示管理员的 ID 为 1，因此可以利用 IDOR 把管理员密码重置为攻击者指定的值，再正常登录管理员账号。

## 解题过程

资料页表单生成：

```php
<input
    type="hidden"
    name="id"
    value="<?php echo $_SESSION['id'];?>"
>
```

隐藏字段只是浏览器展示行为，不是访问控制。POST 处理逻辑直接取用它：

```php
$id = $_POST['id'];
$username = $_POST['username'];
$password = password_hash($_POST["password"], PASSWORD_BCRYPT);

if ($stmt = $conn->prepare(
    'UPDATE users SET username = ?, password = ? WHERE id = ?'
)) {
    $stmt->bind_param('ssi', $username, $password, $id);
    $stmt->execute();
}
```

这里虽然使用了参数化查询，能够防止 SQL 注入，却没有对象级授权检查。注册并登录普通账号后，拦截更新资料的请求，将请求体改为：

```text
id=1&username=admin&password=KnownPassword123
```

服务器会执行等价更新：

```sql
UPDATE users
SET username = 'admin', password = '<KnownPassword123 的 bcrypt 哈希>'
WHERE id = 1;
```

数据库初始化脚本确认 ID 1 的 `is_admin` 为 1。更新语句只改变用户名和密码，不改变该标志。退出普通会话后使用：

```text
username=admin
password=KnownPassword123
```

重新登录，登录逻辑会根据数据库中的 `is_admin` 设置 `$_SESSION['admin']`。访问 `/?page=admin` 后得到：

```text
N0PS{1d0R_p4zZw0rD_r3Z3t}
```

## 方法总结

- 核心技巧：篡改资料更新请求中的用户 ID，把“修改自己的资料”变成“修改管理员资料”。
- 识别信号：敏感对象 ID 来自隐藏字段或 URL，服务端查询按该 ID 执行，但没有与当前会话主体进行绑定。
- 复用要点：参数化查询只解决注入问题，不能替代授权。更新当前用户时应由服务端从会话读取 ID；若确实允许指定对象，也必须在执行查询前做逐对象权限校验。
