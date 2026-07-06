---
type: family
tags: [pentest, family, linux, privesc, service-abuse]
skills: [ctf-pentest]
raw:
  - ../raw/pentest/linux-privesc.md
updated: 2026-06-12
---

# Linux Privilege Escalation and Service Exploitation

## 作用边界

本页是 Linux 单机提权与服务滥用 family。它用于从低权限 shell、受限服务账号、数据库账号、备份文件、sudo 规则、SUID/capabilities、cron/systemd、NFS/SSH socket 和本机代理服务中判断提权路线。

如果问题已经变成多主机横向、域渗透或复杂隧道，应回到 [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)。如果只是需要稳定调用工具，见 [pentest-tooling.md](pentest-tooling.md)。

## 识别信号

- 已有 Linux shell 或可执行命令，但当前用户不能直接读 root flag。
- `sudo -l` 出现通配符、受限命令、可控参数、编辑器、抓包/归档/解析工具或服务管理命令。
- 本机存在 root cron、systemd/Monit/PaperCut/Zabbix/Apache/PostgreSQL/Squid 等服务，且低权用户可影响配置、脚本、数据目录或 API。
- 可读备份、NFS share、Unix socket、数据库文件、客户端配置或加密凭据中可能包含下一跳凭据。
- `id`、`getcap`、SUID、Docker group、ACL、GECOS 字段或可写 PATH 暴露本机权限边界。

## 最小证据

- 当前用户、组、shell、工作目录、可写目录、sudo 规则和关键服务进程。
- 每条候选路线都要有手工证据：文件权限、命令参数、配置路径、可写性、服务运行用户和触发方式。
- 对凭据类发现，至少验证一次登录、数据库连接、API 调用或本地 socket 访问。
- 对 race/TOCTOU 类路线，先用低风险目标证明能稳定切换对象，再碰敏感路径。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| sudo 通配符、glob、可控参数 | 用 `sudo -l` 固定规则，检查通配符是否跨参数、是否跟随 symlink、是否能二次指定输出/配置文件。 | [pentest-tooling.md](pentest-tooling.md) |
| root 服务读取进程命令行或配置 | 证明低权用户能伪造进程名、覆盖配置或影响服务工作目录，再触发 root 执行/读文件。 | [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |
| cron/systemd/备份任务处理可写目录 | 检查复制是否保留 SUID、是否改变 owner、是否执行可写脚本或加载可控文件。 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| PostgreSQL/MySQL/Zabbix 等数据库权限 | 区分数据库内读写、文件读、`COPY TO PROGRAM`/UDF/RCE、密码 hash 提取和应用后台接管。 | [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |
| NFS、备份、SSH key、Unix socket | 先挂载/转发并验证最小访问，再提取凭据或进入受限本地服务。 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| PaperCut、Squid、Apache、Monit 等服务 | 先确认服务用户、监听面、管理 API、配置覆盖顺序和日志/错误输出可读性。 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| SUID、capabilities、Docker group、ACL | 先用 GTFOBins/手工命令证明读文件或 shell 能力；Docker group 可直接等价 root。 | [pentest-tooling.md](pentest-tooling.md) |
| polkit/内核/服务 N-day | 固定版本和补丁状态；能用配置/权限链解决时，不优先跑不确定 exploit。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| `stat`/`access` 后 `open` 的路径检查 | 先验证 symlink swap 或 rename race 是否能打到普通文件，再升级到 root 文件。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |

## 关键路线族

| 路线族 | 可复用判断 |
|---|---|
| sudo/GTFOBins 类 | 重点不是命令名，而是 sudoers 约束如何被 argv、glob、二次 flag、编辑器命令或 symlink 解开。 |
| 服务配置劫持 | Monit/Apache/PaperCut 这类路线要证明 root 进程会读取或执行低权可控内容。 |
| 数据库到系统 | PostgreSQL superuser、备份目录、`pg_authid`、Zabbix MySQL 都可能从数据权限跳到系统或后台权限。 |
| 备份/NFS/Unix socket | 先把不可达资源转成可访问端口或本地挂载，再判断是凭据、文件读还是执行能力。 |
| 常规本机提权 | SUID、capabilities、Docker、ACL、cron、PATH、可写服务脚本仍需手工确认，自动化枚举只是候选生成器。 |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖 sudo 参数注入、服务配置、数据库 RCE、备份凭据、NFS/SSH socket、代理 pivot、CVE 和 TOCTOU，首轮判断不同，不应伪装成单一 technique。
- 不拆成 sudo、数据库、服务、NFS 等多个页面：目前这些材料多来自同一 Linux 后渗透语境，拆分会让查询入口碎片化；若后续某一路线积累到 3 个以上独立 raw 且有不同工具链，再拆 technique。
- 不并入 [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)：本页只处理单台 Linux 从低权到更高权限的路线族。

## 常见陷阱

- 只跑 linpeas/lse，不手工确认 `sudo -l`、SUID、capabilities、cron 和服务 owner。
- 看到 wildcard sudo 就只测试路径，不测试是否能注入额外参数或覆盖输出文件。
- 数据库拿到 shell 后忘记检查备份、hash、应用配置和本地 socket，错过更稳定的 root 路线。
- NFS/SSH socket/代理转发建立后没有最小验证，后续失败时不知道是凭据、网络还是服务问题。
- 对 race 类提权直接碰 `/root/flag`，没有先证明文件替换窗口和命中率。

## 关联技巧

- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [pentest-tooling.md](pentest-tooling.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)
- [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)
- [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)

## 原始资料

- [linux-privesc.md](../raw/pentest/linux-privesc.md)
