# OSQ Revenge

## 题目简述

题目提供一个受限的 `osqueryi` 交互环境，要求从主机工件中重建一条攻击链，并按顺序找出六段 flag：两段初始落点、两段持久化、一段横向移动和一段提权。虽然初始入口是 Web 漏洞，决定性工作是通过进程、文件、授权密钥、ARP 缓存、计划任务和 sudoers 等主机证据完成事件关联，因此归入 Forensics。

## 解题过程

### 1. 初始落点：Apache 页面与 WebShell

查询进程可发现 Apache。受限环境不能任意读取文件，但 osquery 的 `curl` 虚拟表可以访问本机页面：

```sql
SELECT response_body
FROM curl
WHERE url = 'http://127.0.0.1/index.html';
```

页面源码显示上传目录为 `/var/www/html/upl04ds`，上传后的文件名不受限制，`page` 参数又会从该目录执行 `include`，形成“上传 PHP + LFI include”的 RCE 链。HTML 注释给出第一段：

```text
shellmates{W3B_Lf1_2Rc3
```

再枚举上传目录：

```sql
SELECT filename, path
FROM file
WHERE directory = '/var/www/html/upl04ds';
```

可见 PHP WebShell `_4fO0THoLd.php`。其逻辑是读取 `cmd` 参数并调用 `system`，文件名本身就是第二段 `_4fO0THoLd`。

### 2. 持久化：cron 与 SSH authorized_keys

计划任务表显示 `sysadmin` 每分钟执行 `/home/sysadmin/_CR0N&_`：

```sql
SELECT event, command, path
FROM crontab;
```

可执行文件名给出第三段 `_CR0N&_`。

继续关联用户和授权密钥：

```sql
SELECT u.username, a.algorithm, a.key, a.comment, a.key_file
FROM authorized_keys AS a
JOIN users AS u ON a.uid = u.uid
WHERE u.username = 'sysadmin';
```

密钥字段中的 Base64 数据 `JFNIX0tlM3BfVV9Jbgo=` 解码为 `$SH_Ke3p_U_In`，得到第四段。这里不是在破解 SSH，而是在识别被植入的持久化工件。

### 3. 横向移动：异常 ARP 缓存

查询 ARP 缓存可见多条静态映射，其中 `de:ad:be:ef:ff:ff` 是干扰项，`5f:34:72:70:5f:61` 才是编码数据：

```sql
SELECT address, mac, interface
FROM arp_cache;
```

把 MAC 的六个十六进制字节按 ASCII 解码：

```text
5f 34 72 70 5f 61  ->  _4rp_a
```

结合题目关于 MiTM 的提示，这一工件表示通过 ARP 欺骗进行横向流量截获，并给出第五段 `_4rp_a`。

### 4. 提权：sudoers 白名单

最后检查 sudoers：

```sql
SELECT * FROM sudoers;
```

配置允许 `sysadmin` 无密码以 root 身份执行 `/Nd_Sud0_4ur_amB1TiOn}`。命令路径给出最后一段 `Nd_Sud0_4ur_amB1TiOn}`；它与上一段末尾的 `a` 拼接成单词 `aNd`。

按攻击链顺序拼接六段，得到：

```text
shellmates{W3B_Lf1_2Rc3_4fO0THoLd_CR0N&_$SH_Ke3p_U_In_4rp_aNd_Sud0_4ur_amB1TiOn}
```

## 方法总结

- 核心技巧：把 osquery 各表中的离散主机工件按“落点—持久化—横向—提权”时间逻辑串成证据链。
- 识别信号：受限 SQL 取证环境中若不能直接读文件，应检查 `curl`、`file`、`processes`、`crontab`、`authorized_keys`、`arp_cache` 和 `sudoers` 等表能否间接暴露内容。
- 复用要点：题目可能把 flag 编入文件名、Base64 密钥字段、MAC 字节或命令路径；仍要先用攻击语义确认相关性，不能把每个异常字符串都无条件拼接。
