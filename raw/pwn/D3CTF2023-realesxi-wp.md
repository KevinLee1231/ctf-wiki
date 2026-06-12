# RealESXi

## 题目简述

题目假设已经在 ESXi `vmx` 沙箱内获得任意代码执行，但仍受 sandbox profile 限制。目标是利用旧版本 vmx 对 settingsd ticket 目录的读写权限，伪造 ticket 调用 lifecycle/settingsd 服务；再利用 CVE-2021-22043 相关临时文件 TOCTOU，在 `OfflineBundle` 下载重试窗口中把临时文件替换成符号链接，实现任意文件写，最终改写 `inetd.conf` 获得 shell。

## 解题过程

本题是一个真实 ESXi 沙箱逃逸。假设已经能在 `vmx` 中执行任意代码，但仍被限制在沙箱内，目标是逃出沙箱并获取 flag。

下载并安装已修复版本 7.0U3c。对两个版本的 settingsd 做 diff 后可以发现，`TicketGetTicketDir` 函数新增了 `"remoteDevice"` 以及一些相关代码。

仅靠这些信息还不足以定位问题。继续比较两个版本 vmx（globalVMDom）的 sandbox rules，可以发现新增了以下规则：

```
-r /var/run/vmware/tickets       # blocklist
```

- `-r /var/run/vmware/tickets-remoteDevice rw`

- `-r /var/run/vmware/tokend-secret # 阻止列表`

这些规则限制了 vmx 进程对 `/var/run/vmware/tickets` 和 `tokend` - `secret` 的读写权限，同时授予了对 `tickets` - `remoteDevice` 的读写权限。

换句话说，在 7.0U3c 之前的版本中，vmx 文件可以读写 `/var/run/vmware/tickets` 文件夹，该目录保存 settingsd 的认证校验文件。正常情况下，与 settingsd 通信需要输入用户名和密码；但通过伪造 ticket，可以绕过这层验证。

Settingsd 绑定在 8083 端口。我原本计划通过抓包快速理解与 settingsd 的通信机制，但 ESXi 默认没有与 settingsd 相关的可触发功能。此时注意到 settingsd 位于

`/usr/lib/vmware/lifecycle/bin/` 目录下，它可能是 lifecycle 功能的服务进程。VMware vSphere Lifecycle Manager 文档的关键信息是：该组件负责 ESXi 集群/镜像/补丁生命周期管理，会通过 settingsd 相关服务处理 depot、vendor index 和 metadata bundle。安装 vSphere 并创建集群后，可以抓到与 8083 端口交互的真实请求，从而还原 settingsd 的 API、ticket 文件格式和 depot 文件结构。这大幅降低了本题难度，让我们可以结合

`settingsd_vapi_cli.json` 理解数据包结构、ticket 文件以及后续会用到的 depot 文件内容。

既然已经可以访问 settingsd，下一步就是分析 CVE-2021-22043 这个 TOCTOU 漏洞。公告说明该漏洞存在于临时文件处理过程中，可被利用实现任意文件写。对 settingsd 做 diff 后，我没有发现类似漏洞被修复，于是转向 lifecycle 目录中的其他文件，尤其是几个 Python 文件。Settingsd 的很多功能会直接调用这些文件。可惜 diff 这些文件也没有发现明显差异。不过它们会调用 vmware 库（`/lib64/python3.8/site-packages/vmware/`）中的部分内容，所以问题可能在这里。

对 Python vmware 库文件做 diff 后，可以快速定位 bug。`Downloader.py` 中存在差异，更重要的差异来自 `OfflineBundle.py`。

```
fh, fn = tempfile.mkstemp()
os.close(fh)
```

在 7.0.3c 中，代码已经被修补为：

```
with tempfile.NamedTemporaryFile() as f:
```

这与公告描述完全吻合。创建该 tempfile 后，`OfflineBundle` 会调用 `Downloader.Get`，它接受一个 URL，可以是本地路径，也可以是网络地址。如果网络访问不可用，`OfflineBundle` 不会立即退出，而是等待并重试。

至此，完成本题需要的工具链已经齐备。通过伪造 ticket，可以绕过验证并控制 settingsd。随后发送 `depots create` 命令，把本地准备的 depot 文件夹中 `vendor` - `index.xml` 的 `metadata.zip` 设置为我们的网络地址。此时先不要启动准备好的 HTTP server。`OfflineBundle` 会创建临时文件，但由于它无法访问 `metadata.zip`，会等待并多次重试。在这个窗口内，可以删除 tempfile，创建指向任意文件的符号链接，然后再启动 HTTP server。`OfflineBundle` 就会把我们准备的 `metadata.zip` 复制到指定文件。

拿到任意文件写后，下一步是获取 shell。可以参考 f1yyy 在 2018 年的 ESXi VM escape 思路：覆盖 `/var/run/inetd.conf`，把绑定端口的 sshd 命令改成 `sh` - `i`。然后向 inetd 进程发送 SIGHUP，再用 `nc` 连接 sshd 端口，即可完成利用。

## 方法总结

- 核心技巧：ESXi sandbox profile diff、settingsd ticket 伪造、vSphere lifecycle depot 协议还原、`mkstemp` 临时文件 TOCTOU、符号链接任意文件写、改写 inetd 获取 shell。
- 识别信号：补丁版本把某目录从可读写改成 blocklist/单独 remoteDevice 目录时，应检查旧版本沙箱内进程是否能伪造认证材料；临时文件从 `mkstemp` 改为 `NamedTemporaryFile` 往往指向 TOCTOU 修复。
- 复用要点：VMware 文档的作用是确认 lifecycle/settingsd 的业务入口和 depot 格式；真正漏洞利用依赖“下载失败重试期间临时文件路径可被替换”。
