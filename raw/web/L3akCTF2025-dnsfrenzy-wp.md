# L3akCTF 2025 DNSFrenzy Writeup

## 题目简述

题目实现了一个自定义递归 DNS 解析器。普通查询由 UDP 5335 接收，解析器则把对上游 DNS 的请求固定从 UDP 53535 发出。只有与请求来源 IP 绑定的内部子域被缓存为 `127.0.0.1` 时，服务器才会在 DNS TXT 记录中返回 flag。

决定性障碍是组合可预测 TXID、响应来源未校验、竞争窗口和可污染的查询名，本文按 Web 归档。

## 解题过程

### 还原内部域名和 TXID

对来源 IP 为 `caller` 的客户端，允许访问的内部名称为：

```python
import hashlib

subdomain = hashlib.sha256(caller.encode()).hexdigest()[:63]
internal = f"{subdomain}.dns_l3ak.ctf."
```

解析器的 TXID 也取决于来源 IP和当前时间：

```python
timestamp = int(time.time()) // 0.2
data = f"{caller}_{timestamp}".encode()
txid = struct.unpack("!H", hashlib.md5(data).digest()[:2])[0]
```

代码先对 `time.time()` 取整，因此实际只有一秒级时间不确定性，并非真正每 0.2 秒产生一个独立时间桶。远程利用时可同时尝试当前秒及相邻一秒的候选。

### 抢在真实上游之前注入响应

`send_query` 先把 TXID 加入 `pending_tids`，然后故意等待 3 秒才向真实 DNS 服务器发送数据。后台监听线程收到 UDP 包时只检查：

```text
TXID 是否正在等待
该 TXID 是否已有响应
```

它不验证响应源 IP、源端口、问题名或问题类型。攻击者可以在 3 秒窗口内直接向目标 UDP 53535 发送伪造响应。

第一次伪造响应应同时包含：

- question 的 `qname`：攻击者对应的内部域名；
- authority：把该内部域名声明为 NS；
- additional A：把该 NS 指向攻击者可控、能监听 UDP 53 的地址；
- 与待处理请求相同的 TXID。

解析器收到它后会执行：

```python
query.questions[0].qname = response.questions[0].qname
```

也就是说，客户端原本查询的 `google.com.` 被替换为内部域名。下一跳又被指向攻击者的 DNS 服务器，后续权威查询和最终查询都由攻击者回答。

### 让错误名称进入缓存

攻击者的 UDP 53 监听器对后续查询原样复用收到的 TXID，并返回：

```text
question:  <caller_hash>.dns_l3ak.ctf.
answer A:  127.0.0.1
authority: 同一内部域名
additional A: 攻击者 DNS 地址
```

最终查询要求 A 记录时，解析器取出 `127.0.0.1`，随后执行：

```python
self.update_cache(query.questions[0].qname, final_ip)
```

此时 `qname` 已被第一次伪造响应改成内部域名，于是受保护名称被缓存为本地回环地址。

利用过程可以概括为：

```python
# 线程 1：监听攻击者的 UDP 53，回答目标后续发来的 DNS 请求
start_fake_authoritative_dns()

# 线程 2：向目标 5335 查询任意外部域名，触发递归解析
send_query(target, 5335, "google.com.", predicted_txid)

# 线程 3：在等待窗口内向目标 53535 直接注入第一次伪造响应
send_spoof(target, 53535, internal_name, predicted_txid)
```

污染成功后，再从同一来源 IP 向 UDP 5335 查询该内部域名。服务端命中 `127.0.0.1` 缓存，并在答案中附加 TXT 记录：

```text
L3AK{bad_tx1d_+_r4c3_c0nd1t10n_+_dn5_sp00f1ng_=_flag!!!}
```

本地官方脚本把调用方、伪造 DNS 和目标都设为 `127.0.0.1`。部署到远程环境时，内部域名前缀必须基于目标实际看到的客户端 IP；additional A 则应填目标能够访问的攻击者 DNS 地址，两者不一定相同。

## 方法总结

DNS 响应关联不能只依赖 16 位 TXID。解析器至少还应校验源 IP、源端口、question 名称和类型，并使用真正随机的 TXID；本题的时间哈希既可预测，又因固定 3 秒延迟给攻击者留下稳定竞争窗口。

更严重的是，解析器信任响应中的 question，并把修改后的名称写入缓存。这样一次针对普通域名的递归查询就能污染受保护的内部命名空间。安全缓存的键必须来自原始请求，且 delegation 响应只能在正确的域名边界内影响下一跳。
