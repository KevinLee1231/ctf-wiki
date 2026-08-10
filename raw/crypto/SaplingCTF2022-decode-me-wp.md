# Decode Me

## 题目简述

服务随机生成 64 位无符号整数，以四种表示之一输出：8 字节小端序、8 字节大端序后 Base64、Python 风格十六进制或二进制。必须在每轮 30 秒内返回十进制整数，连续完成 1337 轮。难点是自动识别题型并正确处理字节序和表示层编码。

## 解题过程

根据 BEGIN 标题分流。四种解码分别是：

~~~python
import base64

def decode_value(kind, payload):
    if kind == "BYTES (LITTLE ENDIAN)":
        return int.from_bytes(payload, "little")
    if kind == "BASE64":
        raw = base64.b64decode(payload)
        return int.from_bytes(raw, "big")
    if kind == "HEXADECIMAL":
        return int(payload, 16)
    if kind == "BINARY":
        return int(payload, 2)
    raise ValueError(kind)
~~~

字节题可能含不可打印字符，不能用普通按行字符串 API 随意 decode；应按服务的 BEGIN/END 标记读取原始 bytes。Base64 分支先还原 8 个大端字节，再转整数。hex 和 binary 允许 int 的 base 参数直接处理 0x、0b 前缀。

用 pwntools 循环读取编码名称、截取标记间 payload、发送 str(answer)：

~~~python
for _ in range(1337):
    io.recvuntil(b"-----BEGIN ")
    kind = io.recvuntil(b" ENCODED MESSAGE-----", drop=True).decode()
    io.recvline()
    payload = io.recvuntil(b"\n-----END ", drop=True)
    answer = decode_value(kind, payload)
    io.sendlineafter(b"Decoded number: ", str(answer).encode())
~~~

完成所有轮次后得到：

~~~text
maple{15_th15_crypt0??}
~~~

## 方法总结

编码题最常见的错误是混淆“文本中的数字”和“整数的字节表示”，以及忽略端序。自动交互时还要保留原始字节，依据明确边界读取，而不是假定数据都是 UTF-8。把每种格式封装成独立函数并用几个固定整数本地往返测试，可以显著减少长轮次中途失败。
