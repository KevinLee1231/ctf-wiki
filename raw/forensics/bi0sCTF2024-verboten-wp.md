# bi0sCTF 2024 - verboten

## 题目简述

题目给出一份 Windows AD1 镜像和九个问题。故事从插入未知 USB 开始，后续证据分散在 SYSTEM/SAM 注册表、Chrome、启动目录、Slack、Google DriveFS、AnyDesk、Prefetch 和 Windows Timeline 数据库中。题目明确所有时间均按 IST，即 UTC+05:30 表示。

## 解题过程

### USB、下载与持久化

用 FTK Imager 打开 AD1，导出注册表 hive 和用户 `randon` 的应用数据。SYSTEM hive 的 `CurrentControlSet\Enum\USBSTOR` 保存设备实例、序列号和首次/末次连接相关时间。按题目格式组合序列号与 IST 时间：

```text
Q1  verboten{4C530001090312109353&0:2024-02-16-12-01-57}
```

Chrome `User Data\Default\History` 的 `downloads`、`downloads_url_chains` 表显示 USB 触发访问：

```text
https://filebin.net/qde72esvln1cor0t/mal
```

恢复下载内容并计算 MD5：

```text
Q2  verboten{11ecc1766b893aa2835f5e185147d1d2}
```

`mal.exe` 被复制进用户 Startup 目录，因此重启后仍会运行。对持久化副本计算 MD5：

```text
Q3  verboten{169cbd05b7095f4dc9530f35a6980a79}
```

### Slack、DriveFS 与远程控制

Slack 的 IndexedDB 可恢复对话文本，Cache/Code Cache 则保留附件响应。对话中包含 AnyDesk 邀请码，附件缓存中可恢复 ZIP。两项按“ZIP MD5:邀请码”提交：

```text
Q4  verboten{b092eb225b07e17ba8a70b755ba97050:1541069606}
```

Google DriveFS 的：

```text
AppData\Local\Google\DriveFS\110922692857671422467\content_cache
```

保存了五份同步文件的内容块。不能把索引/元数据文件本身当作同步内容；应依据 DriveFS 元数据映射并对重组后的五个文件求 MD5：

```text
Q5  verboten{ae679ca994f131ea139d42b507ecf457:4a47ee64b8d91be37a279aa370753ec9:870643eec523b3f33f6f4b4758b3d14c:c143b7a7b67d488c9f9945d98c934ac6:e6e6a0a39a4b298c2034fde4b3df302a}
```

AnyDesk 的系统级 `ProgramData\AnyDesk\connection_trace.txt` 和用户级 `AppData\Roaming\AnyDesk\ad.trace` 能相互印证入站连接时间与对端 ID。统一转换到 IST 后：

```text
Q6  verboten{2024-02-16-20-29-04:221436813}
```

### 删除工具、SAM 安全问题与剪贴板

Prefetch 中出现 `BLANKANDSECURE_X64.EXE`，说明攻击者曾运行安全删除工具。Prefetch 的 last run time 换算为题目要求时区后为：

```text
Q7  verboten{2024-02-16-08-31-06}
```

SAM hive 的 `SAM\Domains\Account\Users` 下可定位用户记录；其 `ResetData` 包含密码重置问题及回答。提取题目要求的三项：

```text
Q8  verboten{Stuart:FutureKidsSchool:Howard}
```

最后检查 `ActivitiesCache.db`。`ActivityType=10` 的 SmartLookup/剪贴板活动在 `ClipboardPayload` 中保存 Base64 数据；解码可得一次性代码 `830030`。`StartTime` 是 Unix 时间，转换到 IST 后为 `2024-02-16 23:24:43`：

```text
Q9  verboten{830030:2024-02-16-23-24-43}
```

九项均通过后得到：

```text
bi0sctf{w3ll_th4t_w4s_4_v3ry_34sy_chall_b9s0w7}
```

工件原始界面和查询截图可在[官方题解](https://blog.bi0s.in/2024/03/08/Forensics/verboten-bi0sCTF2024/)中核对；正文已把截图中的 URL、路径、时间、标识和全部提交值转写为文本。

## 方法总结

本题考查典型 Windows 终端时间线关联。USBSTOR 确认入口，Chrome 说明下载来源，Startup 证明持久化；Slack 与 DriveFS 恢复协作和同步数据，AnyDesk 证明远程连接，Prefetch 证明擦除工具执行，SAM 与 ActivitiesCache 分别给出安全问题和剪贴板验证码。全程最容易出错的是对象选取与时区：哈希必须针对恢复后的实际内容，所有时间统一换算为 IST 后再按指定格式提交。
