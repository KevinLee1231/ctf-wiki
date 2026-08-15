# Homer The Simp - 0x1

## 题目简述

题目描述 Homer 正在帮助 Marge 通过 SFTP 交换考试文件，并透露了用户名 `sftp`，但没有给出密码。已知目标长期使用同一张“面对电脑”的头像，且主要活动在当时以蓝鸟为标志的 Twitter。目标是通过公开信息找到正确账号、SFTP 密码和本题 flag。

本题的核心证据来自公开账号关联，归入 OSINT 方向。仓库中的部署源码还保留了服务端账号配置，可用于交叉验证公开信息，而不是仅凭题目附带的 flag 反推答案。

## 解题过程

以题目给出的昵称 `HomerTheSimp`、平台特征和头像描述联合检索，可以定位到历史账号 [`@homerthesimp56`](https://twitter.com/homerthesimp56)。不能只凭相似用户名认定目标，还要核对以下线索是否同时吻合：

- 账号头像是题面所说的 Homer 面对电脑；
- 账号内容围绕 Homer 与 Marge；
- 某条推文下的回复直接涉及双方约定的 SFTP 凭据。

该回复泄露了 SFTP 密码：

```text
m@rge1sl1f3
```

比赛仓库中的 Homer 容器构建文件独立印证了这条情报：

```dockerfile
useradd -d /home/homer -s /usr/sbin/nologin sftp
echo 'sftp:m@rge1sl1f3' | chpasswd
```

因此后续连接所需的凭据为 `sftp:m@rge1sl1f3`。同一回复中还给出了本题 flag：

```text
shellmates{w3_c4ught_h1m_s1mp!ng_l4ds}
```

## 方法总结

这类账号定位题应把昵称、平台、头像和内容关系视为一组约束，而不是只搜索一个关键词。公开回复中的密码属于决定性证据；部署源码中的 `chpasswd` 配置则提供了第二条、相互独立的验证链。

历史社交媒体内容可能删除、改名或要求登录，因此题解正文必须记录真正有用的信息。本题即使不再访问外链，也能从正文得知账号、筛选依据、SFTP 凭据以及源码验证位置；保留账号链接仅用于指向原始公开来源。
