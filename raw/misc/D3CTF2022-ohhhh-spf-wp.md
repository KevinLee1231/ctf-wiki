# OHHHH!!! SPF!!!

## 题目简述

题目描述直接给出 L2TP 隧道凭据，并要求选手创建 L2TP tunnel 后启动 OSPFv3 实例来获取 flag：

> \# L2TP Tunnel
>
> User: D3CTF
>
> Password: AFZcByFx5c2dQxXr
>
> IPsec Secret: M99iDSq6RAHY5quU
>
> Create a L2TP tunnel and launch a OSPFv3 Instance to get flag
>
> Server OS: RouterOS v6.49.4 CHR

本题不是传统文件隐写或漏洞利用，而是网络路由协议小题：连上 RouterOS 后与对端建立 OSPFv3 邻接，从收到的 IPv6 路由表中提取编码数据，再按 GBK 解码得到 flag。

## 解题过程

这个题目其实算不上是一个题目

这个题的灵感来自 Soha 的红包小游戏，只是将 BGP 换成了 OSPF，致敬 Soha！

详情可见 Soha 的博客：https://soha.moe/post/find-soha-red-packet-2021.html

Soha 2021 红包题的关键思路是：通过 BGP 建邻后收到若干 IPv6 路由，路由前缀中的十六进制数据按顺序重排并用 GBK 解码，可以得到中文提示和红包口令。本题沿用了“路由表里藏信息”的想法，但把 BGP 改成 OSPFv3，并用 L2TP 隧道把选手接入同一个路由环境。

这只能算是一个整活向的小甜点，因为并不涉及什么安全相关的内容，目标是顺带安利一下 RouterOS:-)

硬要说有什么关联我也只能说可能实战里遇到可以快速得知目标的网络拓扑了

这个题的流程在题目描述里阐述的很清楚了

只需要起一个 L2TP 隧道，再起一个 OSPFv3 实例

收到路由表之后 GBK 编码转换一下就得到 flag 了

L2TP 隧道就不详细说了，下面是常见的几种 OSPFv3 配法。

### BIRDv2

```
protocol ospf v3 {
  tick 2;
  rfc1583compat yes;
  ipv6 {
    import all;
    export all;
  };
  area 0 {
    interface "ppp0" {
      type broadcast;
    };
  };
};
```

### RouterOSv6

```
/routing ospf-v3 instance
add name=D3-OSPF router-id=10.255.255.2
/routing ospf-v3 area
add instance=D3-OSPF name=D3-Area
/routing ospf-v3 interface
add area=D3-Area interface=D3-VPN network-type=point-to-point
```

### RouterOSv7

```
/routing ospf instance
add name=D3-OSPF redistribute=static router-id=\
    10.255.255.3 routing-table=main version=3
/routing ospf area
add instance=D3-OSPF name=D3-Area
/routing ospf interface-template
add area=D3-Area cost=10 interfaces=D3-VPN priority=1 type=ptp
```

## 方法总结

- 核心技巧：按题面凭据建立 L2TP 隧道后，让本机路由软件参与 OSPFv3 邻接，等待对端下发路由。
- 信息提取：收到的 IPv6 路由不是单纯网络拓扑信息，前缀中携带了编码数据；把路由字段按正确顺序取出后用 GBK 解码即可得到 flag。
- 复用要点：网络协议类 Misc 题不要只看应用层文件，BGP/OSPF/RIP 等路由表、LSA、前缀和 next-hop 都可能被用作载体。
