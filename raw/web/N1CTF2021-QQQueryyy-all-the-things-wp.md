# N1CTF 2021 - QQQueryyy all the things

## 题目简述

入口把参数 `str` 直接拼入一条 SQL，再交给 `osqueryi` 执行：

```php
$str = $_GET['str'] ?? 'world';
$sql_query = "SELECT '".$str."' as hello;";
$args = escapeshellarg($sql_query);
system("echo ".$args." | osqueryi --json");
```

`escapeshellarg` 只能保护外层 shell 参数，不能修复 SQL 拼接，因此可以追加任意 osquery 查询。最终目标不是普通数据库数据，而是借助 osquery 的系统虚拟表发现内网 IoT.js REPL，并把 SQL 注入扩展为 SSRF 与代码执行。

## 解题过程

### 确认堆叠查询与 osquery

令 `str` 为：

```sql
world'; SELECT 123; --
```

服务端实际执行：

```sql
SELECT 'world'; SELECT 123; --' as hello;
```

查询 `sqlite_temp_master` 时会出现 `azure_instance_tags` 等普通 SQLite 不具备的表；结合题目镜像中安装的 `osquery_5.0.1`，可以确认后端是 osquery。此后应把这些表当作操作系统枚举接口，而不是传统业务数据库。

### 枚举文件和隐藏服务

先查询根目录：

```sql
SELECT * FROM file WHERE directory='/';
```

结果中有仅 root 可读的 `/flag`，以及带 SUID 位的 `/readflag`。Web 进程不能直接读 flag，所以还需要代码执行。

继续查询端口、进程与 xinetd 配置：

```sql
SELECT * FROM listening_ports;
SELECT * FROM processes;
SELECT * FROM file WHERE directory='/etc/xinetd.d/';
SELECT * FROM augeas WHERE path='/etc/xinetd.d/ctf';
```

这些查询共同给出如下事实：

- TCP `16324` 端口由 xinetd 管理；
- 服务以 UID/GID 1000 运行；
- 实际命令为 `/src/iotjs/build/x86_64-linux/debug/bin/iotjs /src/iotjs/tools/repl.js`；
- 镜像编译 IoT.js 时启用了 `ENABLE_MODULE_NAPI=ON`。

官方说明也给出另一种读配置方式：用 `yara` 表对目标文件逐字节试探。这里 `augeas` 已经能够直接还原 xinetd 配置，无须再做盲读。

### 通过 curl 表访问 IoT.js REPL

osquery 的 `curl` 表能发起 HTTP 请求，可将 URL 指向本机端口：

```sql
SELECT * FROM curl
WHERE url='http://127.0.0.1:16324/'
  AND user_agent='\n\n\n\n\n\n\n\n\n\n{JS_CODE}\n\n\n\n\n\n\n\n\n\n';
```

关键不是普通 SSRF 回显，而是 `user_agent` 中的换行。IoT.js 的 `tools/repl.js` 按行处理输入，前后各填充多行换行后，中间的 JavaScript 会成为一条可执行的 REPL 输入。官方文档有一处把端口写成 `16234`，但 `ctf.xinetd`、端口枚举结果和 Docker 源码均证明正确端口是 `16324`。

### 从 JavaScript 到 SUID readflag

IoT.js 开启了 N-API，可从 JavaScript 加载本地原生模块。可分两次请求完成：

1. 在第一段 JavaScript 中使用 `http.get` 下载恶意 `.node` 模块，并用 `fs.openSync`、`fs.writeSync` 写入 `/tmp`；
2. 在第二段 JavaScript 中执行 `require('/tmp/payload.node')`，让原生模块调用 `/readflag` 或直接执行所需命令。

另一条官方认可的路线是写 `/proc/self/mem`，改写 IoT.js 进程的 PLT 后获得命令执行。两条路线的共同点都是先用 osquery `curl` 表把 JavaScript 注入 REPL，再越过 JavaScript 层获得本地执行能力。

[参赛者复盘](https://hwwg.github.io/2021/12/08/2021nu1lctf/)记录了“下载原生模块、再 `require`”的两阶段 JavaScript；其中公开脚本的写入回调把变量 `c` 误写为 `nc`，复现时应修正为 `fs.writeSync(f, c, 0, c.length)`。该页面没有提供原生模块本体，因此不能把它当作完整可运行 EXP。

仓库没有保留最终原生模块或线上交互记录，但 `source/flag` 与 `source/info.md` 一致给出本题 flag：

```text
n1ctf{3894619c1b94abe1df7fa7948fa5028a5eba3b98408624ebc02163ad72382c39}
```

## 方法总结

本题的决定性链路是“SQL 注入 → osquery 系统枚举 → `curl` 表 SSRF → 换行注入 IoT.js REPL → N-API 原生模块 → SUID `/readflag`”。`escapeshellarg` 并不等于 SQL 参数化；只要不可信数据仍被拼进 SQL，shell 层转义无法阻止 SQL 注入。分析 osquery 题时还要主动查看 `file`、`processes`、`listening_ports`、`augeas`、`yara` 和 `curl` 等虚拟表，它们能把一个数据库入口扩展为主机侦察与网络访问能力。
