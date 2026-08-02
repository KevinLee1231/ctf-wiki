# N1CTF 2021 - funny_web

## 题目简述

应用把 POST 参数 `url` 交给命令行 `curl`，但过滤字母 `l/g`、控制字符、单双引号，并且只有 curl 输出包含当前 Session UUID 时才回显。内网存在 Microsoft SQL Server，`sa` 密码位于本机 `/password.txt`，flag 则保存在 Windows 注册表。

利用链是：用 curl URL glob 在过滤之后生成被禁的 `file` 与 `gopher` scheme，读取提示和密码字典；再手工构造 TDS Prelogin、Login7 与 SQL Batch 数据包，通过 Gopher SSRF 登录 MSSQL 并调用 `xp_cmdshell` 查询注册表。

## 解题过程

### 用 URL glob 绕过 scheme 黑名单

过滤器为：

```php
$blacklist = "/l|g|[\x01-\x1f]|[\x7f-\xff]|['\"]/i";
```

curl 会在真正发起请求前展开方括号范围和花括号集合。字符串 `fi[k-m]e` 本身不含 `l`，却会枚举出 `file`；同理 `[e-h]opher` 会枚举出 `gopher`。因此可读取本地提示：

```text
fi[k-m]e:///hint.txt
```

回显条件要求结果包含当前 UUID。官方解法把真实目标和 UUID 放进同一个 curl glob：

```text
fi[k-m]e:///{hint.txt,<session-uuid>}
```

同样的方法读取 `/password.txt`。提示给出 MSSQL 内网地址、端口 1433、用户 `sa`，以及 flag 位于：

```text
HKEY_LOCAL_MACHINE\SOFTWARE\N1CTF2021
```

不同部署材料中的内网地址分别出现 `10.11.22.13` 与 `10.11.22.9`，利用时应以当前 `/hint.txt` 为准。

### 手工生成 TDS 登录流量

普通 SQL 客户端无法直接穿过这个 SSRF，需要把完整 TDS 字节流放进 Gopher selector。官方 `exp.py` 依次拼接三类包：

1. TDS Prelogin：声明版本、加密选项等固定字段。
2. Login7：填写客户端名、`sa` 用户、候选密码、服务器名和 `master` 数据库，各字符串使用 UTF-16LE。
3. SQL Batch：30 字节头后追加 UTF-16LE SQL。

TDS 7 密码不是明文 UTF-16LE。对每个字节交换高低半字节后异或 `0xa5`；字符的 UTF-16LE 高字节 0 变换后固定为 `0xa5`：

```python
def tds_password(password):
    out = bytearray()
    for b in password.encode("utf-16le"):
        out.append(((b << 4) | (b >> 4)) & 0xff ^ 0xa5)
    return bytes(out)
```

所有偏移、字符长度和包长分别按 TDS 要求使用小端或大端写入，最后对二进制流做百分号编码。

### 让 SSRF 响应通过 UUID 检查

Gopher scheme 本身也不能直接写出，使用：

```text
[e-h]opher://<mssql-host>:1433/_{<session-uuid>,<percent-encoded-tds>}
```

curl 的花括号会依次展开两次请求。第一项把 UUID 带进服务响应，使 `$res` 通过 `strpos($res, $_SESSION['uuid'])`；第二项才发送完整 TDS 会话。

对 `/password.txt` 中每个候选密码，先执行：

```sql
select 'this_is_test_text';
```

响应出现标记字符串时即确认密码。随后发送：

```sql
exec master..xp_cmdshell
  'cmd /c reg query "HKEY_LOCAL_MACHINE\SOFTWARE\N1CTF2021" /s';
```

查询结果会随 TDS 响应回显，其中包含 flag。仓库没有保存命中密码后的真实输出，因此本文不臆造 flag。

## 方法总结

黑名单只检查 curl 展开前的字符串，而 URL glob 在检查后重新生成危险 scheme，形成典型的解释层差异。SSRF 面向非 HTTP 服务时，必须理解目标协议的握手、编码和长度字段；本题还利用同一 glob 同时注入 Session UUID，解决了应用层回显门槛。
