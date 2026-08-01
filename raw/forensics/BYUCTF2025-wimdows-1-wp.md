# Wimdows 1

## 题目简述

题目提供一台受入侵 Windows 服务器的 [Wimdows 虚拟机证据](https://byu.box.com/v/byuctf-wimdows)，要求判断攻击者最初取得 shell 使用的 CVE。系统中部署了 Elasticsearch 1.1.1，并保留 Windows Event Log 与 Sysmon 进程事件。

决定性证据是多个以 `SYSTEM` 身份运行的可疑进程，其父进程均为 `elasticsearch-service-x64.exe`。

## 解题过程

在事件查看器或 EvtxECmd 中筛选 Sysmon Process Create 记录，按父映像聚合。异常命令的 `ParentImage` 指向 Elasticsearch Windows 服务，说明初始执行发生在该服务进程内，而不是普通 PowerShell 钓鱼链。

再从服务文件、进程信息或安装目录确认版本为 1.1.1。Elastic 官方对旧版动态脚本风险的说明指出，1.2.x 之前可通过请求提交脚本，脚本拥有 JVM 能力，能够读取文件或调用 `java.lang.Runtime` 执行系统命令；1.2.x 起默认禁用这类动态脚本。该机制与事件中的 Elasticsearch 父子进程链完全吻合，详见 [Elastic 的 Scripting and Security 说明](https://www.elastic.co/blog/scripting-security/)。

Elasticsearch 1.1.1 对应的远程代码执行漏洞是 CVE-2014-3120，因此 flag 为：

```text
byuctf{CVE-2014-3120}
```

## 方法总结

- 核心技巧：用 Sysmon 父子进程和运行身份定位被利用服务，再用精确版本与漏洞机制交叉验证 CVE。
- 识别信号：服务进程突然生成 shell、下载器或侦察命令，是服务端 RCE 的强证据；不能只按安装软件列表猜 CVE。
- 复用要点：候选漏洞必须同时满足版本范围、漏洞类型和进程证据，三者缺一就只是猜测。
