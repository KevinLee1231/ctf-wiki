# FBI Open The Door!! 4

## 题目简述

本题沿用 `fish.E01`，要求按 `YYYY-MM-DD HH:mm:ss` 格式给出 Windows 系统安装时间。

[原始 E01 检材（百度网盘，提取码 `2vmz`）](https://pan.baidu.com/s/1oo6k9svcSJHaXo-9R5hDJg?pwd=2vmz)

## 解题过程

使用 FTK Imager 从镜像导出：

```text
C:\Windows\System32\config\SOFTWARE
```

在注册表路径

```text
Microsoft\Windows NT\CurrentVersion
```

中读取 DWORD 值 `InstallDate`。可使用 Registry Explorer，也可执行 `chntpw -e SOFTWARE` 后在交互界面进入上述键。检材中的值为：

```text
InstallDate = 1729666240
```

`InstallDate` 是 Unix 秒级时间戳。按题目环境的 Asia/Shanghai 时区转换：

```powershell
[DateTimeOffset]::FromUnixTimeSeconds(1729666240).ToOffset([TimeSpan]::FromHours(8)).ToString('yyyy-MM-dd HH:mm:ss')
```

结果为 `2024-10-23 14:50:40`，并与自动取证报告中的安装时间一致。

最终提交：

```text
0xGame{2024-10-23 14:50:40}
```

## 方法总结

Windows 安装时间可从 SOFTWARE 配置单元的 `CurrentVersion\InstallDate` 获取。转换 Unix 时间戳时必须明确时区；若直接按 UTC 输出，会与题目要求相差 8 小时。自动取证工具适合交叉校验，但原始注册表键和值才是应写入 WP 的决定性证据。
