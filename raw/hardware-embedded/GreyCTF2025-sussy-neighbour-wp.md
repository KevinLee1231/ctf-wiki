# Sussy Neighbour

## 题目简述

这是现场无线网络渗透题。选手使用支持 monitor mode 的 TP-Link WN722N，先从 802.11 管理帧发现隐蔽线索，再对隐藏 WPA2 网络执行离线口令恢复，进入局域网后从未加密 DNS 流量定位摄像头门户。

## 解题过程

将网卡切到 monitor mode 并监听附近管理帧。一个名为 Tom 的设备持续发送定向 Probe Request，所探测的已保存 SSID 为：

```text
I love gr3y_k17713s
```

Probe Request 会在关联前广播已保存网络名，因此这个 SSID 泄露了后续摄像头页面密码提示“who do i love?”的答案 `gr3y_k17713s`。

同时从 beacon／probe 响应识别隐藏 SSID 的目标 BSSID。网络启用了 Protected Management Frames，不能依赖 deauth 强制握手；改为捕获 AP 广播的 PMKID，导出 WPA 哈希后用 `rockyou.txt` 离线字典恢复弱预共享密钥，再正常接入该 WLAN。

接入后抓取 DNS／mDNS 流量，可看到 `camera.home` 指向局域网摄像头服务。若本机解析不到，就把抓到的 IP 与域名加入 hosts：

```text
<camera-ip> camera.home
```

访问 `http://camera.home`，用户名为 `admin`，密码使用先前 Probe Request 泄露的 `gr3y_k17713s`。登录后访问 `/flag` 即可取得现场 flag。

## 方法总结

隐藏 SSID 不等于不可发现，PMF 也只保护部分管理帧，无法修复弱 WPA2 口令。客户端的定向 Probe Request 可能泄露敏感 SSID，局域网 DNS 又会泄露内部服务名。此题需要完整串联“监听管理帧 → PMKID 离线破解 → 加入 WLAN → DNS 定位虚拟主机 → 利用泄露口令登录”，主障碍是无线接口与 802.11 采集，因此归入 hardware-embedded。
