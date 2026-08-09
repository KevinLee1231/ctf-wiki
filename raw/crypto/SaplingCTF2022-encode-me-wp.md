# Encode Me

## 题目简述

服务给出一个 64 位无符号十进制整数，并指定将其编码为 8 字节小端序、8 字节大端序后 Base64、Python 风格十六进制或二进制。必须连续完成 1337 轮。它与 Decode Me 相反，重点仍是区分整数值、字节序和文本表示。

## 解题过程

完整复刻服务端的四个编码函数：

~~~python
import base64

def encode_value(kind, value):
    if kind == "bytes (little endian)":
        return value.to_bytes(8, "little")
    if kind == "base64":
        return base64.b64encode(value.to_bytes(8, "big"))
    if kind == "hexadecimal":
        return hex(value).encode()
    if kind == "binary":
        return bin(value).encode()
    raise ValueError(kind)
~~~

普通 sendline 会在末尾追加换行，正好符合服务端 readline 的读取方式；但小端原始字节自身可能含换行。出题端会重新抽样直到答案中不含 0x0a，因此客户端可以安全地一次发送一行。交互循环从 Return X as Y 中提取整数和编码名称：

~~~python
for _ in range(1337):
    line = io.recvline_contains(b"Return ")
    number = int(line.split(b"Return ", 1)[1].split(b" as ", 1)[0])
    kind = line.rsplit(b" as ", 1)[1].decode().strip()
    io.sendlineafter(b"Encoded number: ", encode_value(kind, number))
~~~

注意 bytes 分支必须固定写满 8 字节；hex 与 binary 分支需要保留 0x、0b 前缀。完成后得到：

~~~text
maple{d1d_y0u_u5e_pwnt00l5?}
~~~

## 方法总结

整数序列化必须同时明确长度、字节序和外层编码。Base64 只是字节到文本的表示，不会自动决定端序；hex(value) 与 value.to_bytes(...).hex() 的格式也不同。最可靠的做法是逐行复刻题目生成函数，而不是凭输出外观猜测。
