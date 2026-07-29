# TPA 02 - 📱

## 题目简述

Peggy 在个人手机上遭遇了冒充公司 IT 部门的短信钓鱼。题目要求从同一组网络/设备证据中找出攻击者手机号和 Peggy 输入的密码，并按：

```text
r3ctf{去掉符号的号码_密码}
```

提交。仓库没有保留体积较大的原始附件，只保留题面；下面依据公开复现中给出的流编号、数据库路径线索和最终证据整理。

## 解题过程

先在 Wireshark 中按 TCP 流逐条检查，或使用：

```bash
tshark -r capture.pcapng -q -z conv,tcp
tshark -r capture.pcapng -q -z follow,tcp,ascii,31
```

TCP stream 31 中可以直接看到 Peggy 在钓鱼页面提交的密码：

```text
l0v3_aNd_peace
```

还不能只凭网络请求确定短信发送者。继续检查 stream 34，可见搜索请求：

```text
GET /search?q=IT+deptment+SMS
```

它把调查方向指向 Android 的 Telephony Provider。对从流量中恢复的设备文件或文件系统包执行字符串定位：

```bash
strings recovered_phone_data | \
  rg "com\.android\.providers\.telephony|mmssms\.db"
```

找到 `com.android.providers.telephony` 数据目录下的短信 SQLite 数据库后，用 `sqlite3` 检查表结构和短信内容：

```sql
.tables
.schema sms
SELECT address, type, date, body
FROM sms
ORDER BY date;
```

记录中出现两个相近号码：

```text
+15555215558
+15555215556
```

需要结合 `type` 表示的收发方向、时间顺序与正文，而不是任选一个号码。`+15555215558` 对应冒充 IT 部门、诱导 Peggy 输入凭据的对话，因此它才是攻击者号码；另一个号码属于无关短信。

按题面要求删除号码中的 `+` 和其他分隔符，再与 stream 31 的密码拼接：

```text
r3ctf{15555215558_l0v3_aNd_peace}
```

流编号、Telephony 数据库线索和候选号码可参考 [R3CTF 2024 TPA 02 复现](https://blog.shenghuo2.top/posts/0bba261/)。本文已补足从网络流到 SQLite 短信表、再以消息语义排除错误候选的证据链。

## 方法总结

本题需要关联两类证据：网络流给出受害者实际提交的密码，Android Telephony 数据库给出钓鱼短信的发送者。只用 `strings` 搜到手机号并不足以定性，必须根据短信方向、时间和正文确认攻击角色。提交前还要严格按题面规范化号码；展示格式中的 `+` 不是 flag 的一部分。
