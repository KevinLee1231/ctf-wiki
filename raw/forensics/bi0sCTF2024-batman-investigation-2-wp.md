# bi0sCTF 2024 - Batman Investigation II: Gotham Underground Corruption

## 题目简述

题目围绕一份 Windows 10 内存镜像展开，共有 14 个连续问题。证据横跨 Thunderbird 邮件、KeePass 内存残留、Windows 安装与时间线、Go 编写的 Exodus 钱包窃取器、Notepad 堆、被破坏的 7z、链上交易，以及来自 macOS 的 Dropbox 配置。每题答案由服务端校验，全部答对后返回 flag。

决定性工作是从不同内存和应用工件中恢复并关联事实，而非单独攻破某个密码算法或区块链协议，因此归入 forensics。

## 解题过程

### 建立内存基线与恢复邮件

镜像对应 Windows 10 x64 19041，可先用 Volatility 3 建立进程、文件、网络和时间线基线。Thunderbird 进程内存及 MIME 片段中能恢复一封 Salvatore Maroni 发给 Hugo Strange 的邮件和附件 `contact.rar`，从而得到前两项：

```text
Q1  hugo-strange_salvatore-maroni
Q2  61ca5520263677e648297e1aefd90343fdd0388f
```

第二项是题目要求的附件摘要。恢复 MIME 时要先按边界拼接完整 part，再做 Content-Transfer-Encoding 解码，不能直接对搜索命中的局部 Base64 字符串求哈希。

### KeePass 主密码与已删除条目

`filescan`、配置残留和进程信息表明系统使用 KeePass 2.53.1。该版本受 CVE-2023-32784 所代表的主密码编辑控件残留问题影响：进程内存可泄漏除首字符外的大部分字符位置。对候选首字符逐一补全，并实际尝试打开 KDBX，得到密码：

```text
di&YéQ)8h<)1T2j
```

题目提交的是版本与密码的 MD5，而不是明文密码：

```text
Q3  keepass_2.53.1:4e896dbb5120c2c33e0965174c6036c2
```

打开数据库后检查回收站/已删除条目，可恢复 `Azr43lKn1ght` 的密钥：

```text
Q4  Azr43lKn1ght:sw1zwJjvpqVvFNANws1v
```

### 定位 Exodus 与 Go 窃取器

注册表卸载项、Prefetch 和 `$MFT` 时间线共同显示 Exodus 2024-01-07 08:39 被安装：

```text
Q5  exodus-2024:01:07:08:39
```

桌面上的 `services.exe` 并非系统服务。逆向其 Go 运行时入口后可以看到，它遍历 `%USERPROFILE%` 下的 Exodus 钱包目录，将每个文件逐字节与 `0x33` 异或、打包为 ZIP，再连接 `172.30.145.189:6969` 发送：

```text
Q6  services.exe_172.30.145.189:6969
```

Notepad PID 5372 的堆/VAD 中留有操作者记下的压缩包密码 `w&31D,%4> C6'nT`。按题目要求提交其 MD5：

```text
Q7  63eeb3a466f8bab52b44e182e919b512
```

题目给出的 `unknown.data` 与内存中被挖空的 7z 仍保留互补结构。参照 7z 头和可用区段修复归档，再用上面的密码解出两个 `.seco` 文件。由于窃取器只做 `byte ^ 0x33`，再次异或即可恢复原文件；两份原文件的 SHA-1 为：

```text
Q8  3ba6011f5e1b6941f893df6579f87f1c823e7493:5e38ffd5e6d6115a6f4bb21b5216c37392eb544f
```

### 恢复钱包并核对链上记录

Exodus 密码以 URL 编码形式残留在内存中，解码后为 `yb#uO7&n%$£6[E310`。提交其 MD5：

```text
Q9  93bbca2406298b9cb574ac06abf113e4
```

用恢复出的钱包文件和密码打开钱包，再按题目指定账户核对交易历史：

```text
Q10 edward_nigma:eth:15.31:0xab42fc3fb81fca7bec9e49058d80dc928ed5de2e780a54de41d3c1a8587c4ab7
Q11 oswald_cobblepot:eth:13.65:0xcfb6b082B7296FaA5aD18695e1F554AFC23724eD
```

这里交易哈希、地址、资产类型和数额都应从恢复后的钱包状态或公开链数据交叉核验；仅在内存字符串中搜索到某个地址不足以证明它对应题目要求的交易。

### 解开 macOS Dropbox 工件

`contact.rar` 中还包含 Dropbox 的 macOS 工件。旧版文件后备 keystore 的 `hostkeys` 载荷使用 AES-128-CBC，密钥由文件 inode 派生：

$$
K=\operatorname{MD5}(\text{"ia9"}\mathbin\|\operatorname{str}(inode)\mathbin\|\text{"Xa|ui20"}).
$$

IV 固定为题解中恢复的 16 字节常量。遍历合理 inode 区间，对载荷去掉版本字节与尾部认证数据后解密；明文中出现 keystore 字段 `Client` 时即找到正确 inode。随后从 `hostkeys` 取出 user key `e747b9c8cc2f80b88c6b511f26682b66`，按 Dropbox 版本 0 的 PBKDF2 参数派生 database key `89839f76862bd89b369d99f41ca51cd7`。解密 `config.dbx` 后得到邮箱、用户名和 EC 私钥。最终三项为：

```text
Q12 89839f76862bd89b369d99f41ca51cd7:twoface2681988@gmail.com
Q13 harvey_dent:MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgJGmnt3gVtSX2iswJYf9S3I5X4vPaS1xKHAvFdN5A2tWhRANCAASO63HNoO6rpC9Gfpi5tR+F5kEiRoTzbV/xWscN9McLCTlj2gWtZ+LSrkQkuaO5/XYiHXxCF/1vRWuNspXwY/K9
Q14 e747b9c8cc2f80b88c6b511f26682b66
```

依次提交 14 项后得到：

```text
bi0sctf{b4tm4n_und3rgr0und_inv3st1g4t10n_Succ3ssful_f98a12405d41746836bb4e7df69e44f6}
```

各工件的原始截图、调试输出与 Dropbox 兼容代码可在[官方题解](https://blog.bi0s.in/2024/02/27/Forensics/BatmanInvestigationII-GothamUndergroundCorruption-bi0sCTF2024/)中核对；正文已经保留全部答案及其证据来源，不依赖外链才能复现主线。

## 方法总结

这是一道多工件关联题。稳定的做法是先建立统一时间线，再按“邮件给入口—KeePass 给凭据—恶意程序解释数据如何被改写—堆内存给归档密码—钱包工件验证交易—Dropbox keystore 恢复身份”的顺序推进。每个哈希都应对题目真正要求的对象计算：附件、明文密码、修复并反异或后的原文件不能混淆。最后的 macOS 阶段则利用 inode 派生的 AES 密钥恢复 `hostkeys`，再沿 Dropbox 自身的密钥层级解开数据库。
