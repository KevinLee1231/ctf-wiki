# Real_VMPWN

## 题目简述

本题是 RealWorld 虚拟机逃逸题，目标是从 VMware guest 侧通过 `vmnet-dhcpd` 的 DHCP 协议处理漏洞影响 host 侧进程，最终在宿主机执行指定程序读取 flag。WP 正文保留了 DHCP lease/uid 生命周期、题目泄露点、double free 布局和最终 ROP 思路；外部链接只作为题目来源和复现参考。

## 解题过程

相关链接包括微信文章 <https://mp.weixin.qq.com/s/8nb5Y0CqMPT9cXkt1VpWTw>、复现文章 <https://ama2in9.top/2021/03/17/d3ctf/>、公开 exp 仓库 <https://github.com/WhooAmii/Antctf-vmpwn> 和 VMware 安全公告 <https://www.vmware.com/security/advisories/VMSA-2020-0004.html>。下面直接整理可确认的利用主线。

题目基于 VMware Workstation/Fusion 的 `vmnetdhcp` UAF 漏洞 `CVE-2020-3947` 改编。VMware 安全公告确认该漏洞可导致从 guest 到 host 的代码执行或使 host 侧 `vmnetdhcp` 服务拒绝服务。题目补丁主要围绕 DHCP lease/uid 处理逻辑做改动：一类检查被 patch 掉，使特定 lease 拷贝和释放路径可达；另一处变量初始化被去掉，用于泄露后续利用需要的地址信息。

分析入口是 DHCP 协议和 lease 结构。guest 侧构造 DHCP `REQUEST`、`RELEASE`、`INFORM` 等报文，host 侧 `vmnet-dhcpd` 在处理租约时会经过类似 ISC DHCP 的 lease 维护逻辑。漏洞点在旧 lease 向新 lease 拷贝时对 `uid` 指针生命周期处理不严，释放后的 uid 仍可能留在哈希表/lease 链上；后续再次取出并拷贝该 lease 时，可以把 UAF 进一步转成 double free 和可控堆写。

利用主线如下：

1. 通过题目保留的未初始化变量泄露通道，从 ICMP 响应中取得栈地址和 `vmnet-dhcpd` 程序基址。
2. 发送多组 DHCP `REQUEST` 占位，构造稳定的 lease 与 uid 小堆块布局。
3. 使用多个 `INFORM` 包清理对应大小的 tcache/free chunks，降低后续申请落点的不确定性。
4. 发送 `RELEASE` 触发 uid 释放，再发送 `REQUEST` 让旧 lease 被拷贝并再次释放，形成 double free。
5. 借助后续 `INFORM` 的可控内容把堆写导向已泄露的栈地址附近，布置 ROP。
6. 程序返回或退出路径触发 ROP，调用 `system("/gflag")` 读取 flag。

公开 exp 里需要根据实际 VMware NAT 网卡、guest IP、server IP 和 glibc/tcache 状态调整字段；这些属于复现环境参数，不应写死到通用 WP 中。

## 方法总结

本题的复盘重点是把“协议状态机”和“堆状态机”对应起来：每一种 DHCP 报文会触发哪些 lease 分配、释放、哈希表插入或删除，决定了 UAF 能否变成 double free 和可控写。单看 CVE 描述只能知道 `vmnetdhcp` 有 UAF，真正的 CTF 利用还要借助题目额外泄露、tcache 布局和 raw packet 构造。

复现时建议先用 Wireshark 观察 `bootp`、`icmp` 和 NAT 网卡流量，确认泄露包和 DHCP 报文顺序正确，再调堆布局。由于公开 exp 依赖特定网卡名、IP 段和较旧 glibc/tcache 行为，迁移环境时应优先替换环境字段并重新验证每一步 lease/uid 堆块变化，而不是直接运行原脚本。

