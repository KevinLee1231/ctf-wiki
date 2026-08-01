# BYUCTF 2022 - Alpine 1

## 题目简述

系统管理员已修改 `mjohnson` 的密码，但攻击者仍能通过 SSH 登录。目标是在 Alpine Linux 磁盘证据中找出维持访问所依赖的完整文件路径。

## 解题过程

本地赛事仓库只保存题面与官方结论，原始磁盘镜像未纳入 Git；题面给出的 [Box 附件](https://app.box.com/s/mi71hnua1osbnaludkxvmnbj1p65bi66/file/951434445088) 是三道 Alpine 题共用的证据源。挂载镜像后，应先检查用户目录、shell 历史和 SSH 配置，而不是只查系统密码文件。

在 `mjohnson` 的 `.bash_history` 中可以看到攻击者创建密钥并把**公钥**写入该用户的 `authorized_keys`。需要纠正官方旧文档的一处表述：`authorized_keys` 保存的是获准登录的公钥，不是攻击者私钥。密码被修改不会撤销公钥认证，因此持有对应私钥的攻击者仍可登录。

持久化文件的完整路径为：

```text
/home/mjohnson/.ssh/authorized_keys
```

提交：

```text
byuctf{/home/mjohnson/.ssh/authorized_keys}
```

## 方法总结

密码轮换只影响口令认证，不会自动清理 SSH 信任。调查“改密后仍能登录”时，应核查 `authorized_keys`、文件权限、修改时间与命令历史，并准确区分服务器上的公钥和攻击者持有的私钥。
