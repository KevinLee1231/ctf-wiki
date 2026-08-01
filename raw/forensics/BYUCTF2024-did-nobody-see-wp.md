# Did Nobody See?

## 题目简述

题目提供 Windows 取证镜像，要求找出 VPN 连接使用过的 DNS 服务器。浏览历史已被擦除，但网络接口配置会持久化在 `SYSTEM` 注册表配置单元中。

## 解题过程

加载 `SYSTEM` hive，并进入：

```text
ControlSet001\Services\Tcpip\Parameters\Interfaces
```

逐个检查 GUID 命名的接口子键。普通 DHCP 接口通常没有静态 `NameServer` 值，而 VPN 创建的虚拟接口留下了：

```text
NameServer = 162.252.172.57,149.154.159.92
```

仓库中的 `answer.png` 只是 Registry Explorer 结果截图，路径和值已完整转写，故不再保留图片。提交任意一个有效地址均可，例如：

```text
byuctf{162.252.172.57}
```

另一可接受答案是 `byuctf{149.154.159.92}`。

## 方法总结

网络取证不要只依赖浏览器记录。Windows 的接口级 DNS 设置位于 `SYSTEM` hive；VPN 虚拟网卡往往比已删除的用户层历史留下更稳定的配置痕迹。若存在多个 `ControlSet`，还应结合 `Select\Current` 判断活动控制集。
