# just wait

## 题目简述

题目给出一份约 28 MB 的 PCAPNG，主体是针对 `/login` 的 HTTP SQL 注入流量。大量请求来自 SQLMap，形成明显噪声；真正的外泄请求使用时间型 SQL 注入，把数据库密码某一位置字符的 Unicode/ASCII 数值直接作为 `SLEEP` 秒数。因此 flag 不在响应正文里，而编码在相邻请求的时间间隔中。

## 解题过程

### 从 SQLMap 噪声中找出自定义载荷

先查看 HTTP 请求的 User-Agent，可以发现大量 `sqlmap/1.9.4#stable` 请求。排除这些请求后，剩余流量中出现类似的 URL 解码结果：

```sql
/login?username=username' AND /*AND*/slEeP/*AND*/(
  (SELECT unicode(SUBSTR(password,21,1)))
)--&password=
```

混合大小写和注释只是为了绕过过滤。核心表达式取 `password` 第 21 个字符的数值，并让数据库休眠同样的秒数。攻击脚本按字符顺序重复请求，所以相邻请求时间戳之差就是被外泄字符的整数编码。

### 提取时间戳并解码

官方解法把分析范围收窄到相关帧，排除 SQLMap User-Agent，只保留 HTTP 请求时间：

```bash
tshark -r capture.pcapng \
  -Y '!(http.user_agent contains "sqlmap/1.9.4") and http.request and frame.number >= 95960 and frame.number <= 96851' \
  -T fields -e frame.time_epoch
```

由于这里的休眠值是整数秒，可取时间戳的整数部分并去除连续重复值。随后计算相邻差值并转字符：

```python
with open("time.txt", "r", encoding="utf-8") as f:
    timestamps = [int(line.strip()) for line in f if line.strip()]

flag = "".join(
    chr(current - previous)
    for previous, current in zip(timestamps, timestamps[1:])
)
print(flag)
```

其中 `time.txt` 应是上述 `tshark` 输出截取整数秒、再按出现顺序去除重复值后的结果。解码得到：

```text
shellmates{WAIt_f4evER}
```

仓库中的 `sqlmap.png` 只是 Wireshark 界面中 User-Agent 的文本截图；关键过滤字段、载荷和命令已经完整转写，因此没有复制这张纯文本截图。

## 方法总结

- 核心技巧：从 PCAP 中识别时间型 SQL 注入，把相邻请求的延迟差还原为字符编码。
- 识别信号：响应内容始终相近，但请求载荷含 `SLEEP(字符数值)`，且请求间隔出现几十秒到百余秒的离散值时，应检查时间隐蔽通道。
- 复用要点：先排除自动化扫描噪声，再固定流、帧范围和请求方向；若延迟不是整秒，应该基于请求—响应配对和容差取整，而不能机械截断时间戳。
