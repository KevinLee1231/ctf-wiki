# Records

## 题目简述

题目暗示 flag 藏在 `cyberjousting.com` 的 DNS 记录中。目标不是扫描网站内容，而是枚举常见主机名的 TXT 记录并识别编码文本。

## 解题过程

先查询根域和常见子域：

```text
dig TXT cyberjousting.com
dig TXT www.cyberjousting.com
```

比赛时 `www.cyberjousting.com` 的 TXT 响应中出现一段 Base64：

```text
Ynl1Y3Rme0RONV9SM2Nvbl9NNDV0M3J9
```

解码：

```python
import base64
print(base64.b64decode("Ynl1Y3Rme0RONV9SM2Nvbl9NNDV0M3J9").decode())
```

输出：

```text
byuctf{DN5_R3con_M45t3r}
```

DNS 区域会随运维更新，若当前查询已没有该 TXT，应以比赛期间的官方仓库记录为历史证据，而不是假定命令或 Base64 解码有误。

## 方法总结

DNS OSINT 要覆盖记录类型和子域两个维度；TXT 常用于验证、策略和临时信息，容易藏入编码载荷。发现高熵字符串后先确认字符集与填充，再离线解码并保留原始响应值。
