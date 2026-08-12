# DownUnderCTF 2021 - Sharing is Caring

## 题目简述

题目只给出用户名 `alex_elgato93`，要求通过其公开社交媒体活动定位最近的街道。解题链需要先做跨平台账号关联，再从帖子截图中提取网络标识，最后使用公开 Wi-Fi 地理数据库把标识映射为位置。

## 解题过程

以精确用户名搜索多个平台，可在 Twitch、Twitter、Pinterest 和 Steam 找到使用相同头像与身份线索的账号，并归并到虚构人物 `Alexandros Elgato`。Steam 的历史用户名进一步泄露了新 pivot：

```text
alexandros-elgato
```

用这个新用户名继续搜索，可关联到 GitHub 和 Reddit。检查 Reddit 发帖时，有一张 Kali 虚拟机截图同时暴露了 ARP 命令输出，其中包含 MAC 地址：

```text
DE:73:2C:6C:1B:C1
```

把该地址作为 BSSID 提交给 [WiGLE 无线网络地理数据库](https://wigle.net/)，地图记录把对应接入点定位到澳大利亚新南威尔士州的一处位置。放大地图并检查最近道路，得到：

```text
Charles McIntosh Pkwy
```

将空格替换为下划线后，判题接受缩写或完整 `Parkway`：

```text
DUCTF{charles_mcintosh_pkwy}
DUCTF{charles_mcintosh_parkway}
```

赛事最初曾把街道姓氏误设为 `Mckintosh`，上线约 30 分钟后已修正；当前仓库配置以 `McIntosh` 为准。

## 方法总结

这条链路的关键是逐步扩大而非一次搜索到底：稳定用户名与头像用于第一次账号归并，历史用户名产生第二组账号，截图中的 BSSID 再转入地理数据库。处理公开截图时应检查终端、地址栏、通知和网络命令等边缘信息；用 WiGLE 定位后还要回到地图核对最近道路，并以判题配置确认拼写与缩写。
