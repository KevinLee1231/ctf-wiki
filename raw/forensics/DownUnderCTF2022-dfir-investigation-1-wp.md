# DownUnderCTF 2022 DFIR Investigation 1 Writeup

## 题目简述

题目给出一份 Windows 取证镜像，要求找出攻击者安装的 C2 持久化任务下一次执行的时间，以及 PowerShell stager 连接的 C2 IP，flag 格式为 `DUCTF{hh:mm_IP}`。

关键证据不是普通计划任务，而是 WMI 永久事件订阅。此机制由三类对象组成：

- `__EventFilter`：规定触发条件；
- `EventConsumer`：保存触发后执行的命令；
- `__FilterToConsumerBinding`：将过滤器与消费者绑定。

离线镜像中的 WMI CIM 数据库存放在 `C:\Windows\System32\wbem\Repository\OBJECTS.DATA`。

## 解题过程

### 解析 WMI 永久事件订阅

先从镜像中提取 `OBJECTS.DATA`，再使用能够解析旧版 WMI Repository 的持久化枚举工具，例如 `PyWMIPersistenceFinder.py`：

```powershell
C:\Python27\python.exe .\PyWMIPersistenceFinder.py .\OBJECTS.DATA
```

枚举结果中存在名为 `Updater` 的过滤器，其查询语句为：

```sql
SELECT * FROM __InstanceModificationEvent WITHIN 60
WHERE TargetInstance ISA 'Win32_LocalTime'
  AND TargetInstance.Hour = 12
  AND TargetInstance.Minute = 38
GROUP WITHIN 60
```

`Win32_LocalTime` 的小时与分钟条件直接给出触发时间，因此任务每天在 `12:38` 触发。`WITHIN 60` 表示轮询间隔，不应误读成执行时间。

### 解开 PowerShell stager

与该过滤器绑定的 `EventConsumer` 中嵌入了 PowerShell Empire stager。载荷使用了两层 Base64，并在每层之间以 UTF-16LE 编码。可靠的解码顺序是：

1. 对外层字符串执行 Base64 解码；
2. 按 UTF-16LE 解释文本；
3. 从结果中提取较长的内层 Base64 字符串；
4. 再次 Base64 解码并按 UTF-16LE 解释。

最终载荷中出现连接地址：

```text
http://192.168.0.27:7777
```

题目只要求 IP，因此取 `192.168.0.27`。与触发时间组合得到：

```text
DUCTF{12:38_192.168.0.27}
```

## 方法总结

这道题考查 Windows WMI 永久事件订阅取证。分析时应把过滤器、消费者和绑定关系一起恢复：过滤器回答“何时触发”，消费者回答“执行什么”。面对 PowerShell stager，不要只凭截图判断，应按编码层次逐层解码，并从最终明文配置中提取网络指标。
