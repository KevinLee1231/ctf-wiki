# SUCTF2026-EzRouter

## 题目简述

题目模拟了一套路由器固件管理系统。HTTP 服务负责登录和路由，请求进入各个 CGI 后，再通过 IPC 交给常驻后台进程 `mainproc` 处理。真正需要利用的并不是 CGI 自身，而是 `mainproc` 中保存的 VPN 配置对象：一个恰好填满的 `pass` 字段会让 `strcpy` 越界写入一个 `\x00`，从而改坏紧随其后的 `custom` 指针；随后 `Edit_VPN_Custom` 会沿这个伪造指针写数据，可以进一步覆盖 VPN 的回调函数。

此外，`mainproc` 启动时主动把一段堆内存改为可读、可写、可执行。利用链因此可以落在：

```text
登录绕过
  -> IPC 堆风水
  -> 单字节 NUL 改写 custom 指针
  -> Edit_VPN_Custom 覆盖回调
  -> Apply_VPN 调用 jmp rdi
  -> VPN 对象内跳板
  -> 相邻 custom 堆块中的 shellcode
  -> 将 flag 写入 ./FILE
  -> download.cgi 下载结果
```

官方仓库中的利用脚本主要给出了堆布局和 VPN 对象内跳板；总 WP 中的利用则补全了 `edit`、`apply` 和回显阶段。下面以源码结构为准，把两部分串成一条完整利用链。由于比赛环境中的 flag 是动态内容，仓库固件内的占位文件不能作为最终 flag。

## 解题过程

### 1. 绕过登录并确认进程边界

`src/www/http.h` 对登录请求的判断只检查查询串中是否同时出现 `action=login` 和 `auth=1`。满足条件后，服务端直接创建 session 文件并下发 `session_id`，没有核对用户名或密码：

```http
GET /www/http?action=login&auth=1 HTTP/1.1
Host: target
```

后续访问 CGI 时带上该 cookie 即可通过 `check_auth`。这里要区分两个进程层次：

- `vpn.cgi`、`wifi.cgi`、`list.cgi` 负责解析 HTTP 参数并封装 IPC 消息；
- `mainproc` 是常驻进程，负责保存配置对象并执行 `Set_VPN`、`Edit_VPN_Custom`、`Apply_VPN` 等操作。

因此，多次 HTTP 请求虽然会启动不同 CGI，但产生的堆对象都留在同一个 `mainproc` 堆中，黑名单、WiFi 和 VPN 请求可以组合成堆风水。

### 2. 还原 VPN 对象布局

CGI 发给后台的 `vpn_recv` 大致包含以下连续字段：

```c
char action[0x20];
char name[0x20];
char proto[0x20];
char server[0x30];
char user[0x20];
char pass[0x20];
char cert[8];
char gap;
char custom[0x3000];
```

`mainproc` 中真正落堆的 `vpn_config_req` 大小为 `0xf0`，关键偏移如下：

```text
+0x00  uint16_t custom_len
+0x08  cert[8]
+0x10  apply_cb
+0x18  action[0x20]
+0x38  name[0x20]
+0x58  proto[0x20]
+0x78  server[0x30]
+0xa8  user[0x20]
+0xc8  pass[0x20]
+0xe8  custom_ptr
```

`Set_VPN` 的核心逻辑可以概括为：

```c
vpn = malloc(0xf0);                    // 实际 chunk 大小 0x100
vpn->custom_len = strlen(input->custom);
vpn->custom_ptr = malloc(vpn->custom_len + 1);
vpn->apply_cb = default_vpn_apply;

strcpy(vpn->pass, input->pass);        // 漏洞点
memcpy(vpn->cert, input->cert, 8);
memcpy(vpn->custom_ptr,
       input->custom,
       vpn->custom_len + 1);
```

JSON 字符串提取函数最多复制 `max_len` 个字节。输入恰好为 `0x20` 字节时，`pass` 缓冲区中没有结尾 NUL；后端再执行 `strcpy`，便会继续从源结构的 `cert` 读取。令 `cert[0]` 为 `\x00`，这个终止符最终写到目标 `pass` 之后一字节，也就是 `custom_ptr` 的最低字节。

这不是任意长度溢出，而是一个受堆地址布局约束的 off-by-null：

```text
原 custom_ptr = ...f0
清零最低字节 = ...00
目标 apply_cb = VPN 对象基址 + 0x10 = ...00
```

只要让 VPN 对象用户区基址低字节为 `0xf0`，其相邻 custom 堆块就从 `B+0x100` 开始，地址低字节仍为 `0xf0`；清零后恰好得到 `B+0x10`，即 `apply_cb` 槽位。

### 3. 用 IPC 分配稳定堆布局

`list.cgi` 和 `wifi.cgi` 都能让 `mainproc` 产生可控大小的持久分配。典型步骤是：

1. 添加若干黑名单项；
2. 保存一条 WiFi 配置；
3. 再创建 VPN 对象及其 custom 缓冲区。

源码中 WiFi 请求对应约 `0x90` 大小的 chunk，MAC 项对应约 `0x40` 的 chunk；源码注释中个别尺寸与实际结构不符，利用时应以结构大小和调试结果为准。官方脚本与总 WP 使用的黑名单数量也不同，说明该数量不是漏洞常量，而是特定固件构建和初始堆状态下的布局参数。

实战中应在本地固件中确认：

- VPN 对象的用户地址是否以 `...f0` 结尾；
- custom 堆块是否紧随其后，位于 `B+0x100`；
- off-by-null 后 `custom_ptr` 是否等于 `B+0x10`。

若布局失败，可以通过固件提供的重启接口重置 `mainproc`，再调整黑名单分配数量。不要把某个固定次数当成跨版本通用结论。

### 4. 同时构造长度字段、证书跳板和 custom shellcode

`vpn.cgi` 支持在 `custom` 前添加 `B64:`。解码器会原地写回二进制数据，但不会依据解码后长度额外补 NUL；所以官方脚本在有效数据后显式加入 `\x00`，使 `strlen` 在预期位置停止。

令解码后的 custom 在前 `0x7eb` 字节中不含 NUL，并在其后追加终止符：

```python
custom = shellcode.ljust(0x7eb, b"\x90") + b"\x00"
cert = b"\x00\xe9\xf2\x00\x00\x00\x00\x00"
password = b"P" * 0x20
```

这三个字段承担不同作用：

1. `custom_len == 0x07eb`，以小端序存入对象开头后，前两个字节正是 `eb 07`，即 `jmp short +7`；
2. `cert[0] == 0` 为 `strcpy` 提供终止符，负责清零 `custom_ptr` 最低字节；
3. `cert[1:]` 从对象 `B+9` 开始形成 `e9 f2 00 00 00`，会从 `B+0x0e` 跳到 `B+0x100`；
4. `B+0x100` 正是相邻 custom 堆块，里面放置真正的 shellcode。

控制流跳转关系可直接按偏移验证：

```text
B+0x00: eb 07                 -> B+0x09
B+0x09: e9 f2 00 00 00       -> B+0x100
B+0x100: shellcode
```

总 WP 使用了约 `0x3eb` 的另一组长度和跳板参数；它反映的是另一份利用布局。这里采用官方脚本中与当前源码对象大小直接对应的 `0x7eb` 方案，不能混用两套偏移。

### 5. 把 Edit 操作变成回调覆盖

`Set_VPN` 完成后，off-by-null 已让 `vpn->custom_ptr` 指向 `vpn->apply_cb`。`Edit_VPN_Custom` 没有验证这个指针是否仍指向原 custom 堆块，只按以下方式写入：

```c
len = min(msg->data_len, vpn->custom_len);
memcpy(vpn->custom_ptr, payload, len);
```

因此，一次短的 `edit` 请求就能覆盖回调函数指针的低字节。选择主程序文本段内的 `jmp rdi` gadget，只需要改低两字节，高位仍沿用原回调地址：

```python
patch = p16(jmp_rdi_offset & 0xffff)
edit_custom = b"B64:" + base64.b64encode(patch)
```

总 WP 对应构建使用的低两字节是 `0xbc2f`，官方仓库旧脚本注释中出现过另一组值。gadget 偏移受二进制版本影响，必须从实际 `mainproc` 中重新确认，不能照抄常量。

### 6. Apply 触发与可执行堆

`mainproc` 构造阶段先申请约 `0xf000` 字节，再把对齐后的 `0x21000` 字节堆区设为 RWX，最后释放临时块。因此后续落在该区域内的 VPN 和 custom 分配可直接执行。

`Apply_VPN` 最终调用：

```c
vpn->apply_cb(vpn);
```

System V ABI 下第一个参数位于 `rdi`。回调被部分覆盖为 `jmp rdi` 后，执行流进入 VPN 对象开头，再依次经过 `EB 07`、cert 中的近跳转，最终到达相邻 custom 堆块中的 shellcode。

触发顺序必须保持为：

```text
Set_VPN   -> 放置对象、跳板和 shellcode，并改坏 custom_ptr
Edit_VPN  -> 借伪造 custom_ptr 覆盖 apply_cb
Apply_VPN -> 调用已被改写的回调
```

### 7. 通过下载接口取回 flag

官方脚本的 shellcode 执行 `/bin/sh -c`，把 flag 内容写到 Web 目录；总 WP 使用了更直接且与源码一致的回显路径：让 shellcode 读取 `/app/flag`，并把带有标记的内容写入当前目录的 `./FILE`。

`download.cgi` 在鉴权后固定打开 `./FILE`，不接受用户传入路径。因此 shellcode 执行完毕后，请求：

```http
GET /cgi-bin/download.cgi HTTP/1.1
Host: target
Cookie: session_id=...
```

返回体就是写入的标记和动态 flag。标记可以用来区分真正的利用结果与未命中堆布局时下载到的旧文件。完整利用流程如下：

```python
# 伪代码：省略目标地址、重试与网络包装
sid = login_bypass()
heap_feng_shui_with_list_and_wifi()

set_vpn(
    password=b"P" * 0x20,
    cert=b"\x00\xe9\xf2\x00\x00\x00\x00\x00",
    custom=b64(shellcode.ljust(0x7eb, b"\x90") + b"\x00"),
)
edit_vpn(custom=b64(p16(jmp_rdi_low16)))
apply_vpn()
flag = download_file()
```

## 方法总结

本题的关键不是笼统的“结构体不一致”，而是四个能够逐项验证的原语：恰好 `0x20` 字节的 `pass` 让 `strcpy` 继续读取 `cert`，`cert[0]` 产生 off-by-null，特定堆布局让清零后的 `custom_ptr` 指向 `apply_cb`，而未校验指针的 Edit 操作把这一指针错位升级为回调覆盖。

利用时应把“固定机制”和“版本参数”分开：对象偏移、`custom_len` 小端跳板、`Apply_VPN(vpn)` 令 `rdi=vpn` 属于源码机制；黑名单分配次数和 `jmp rdi` 低两字节则依赖具体构建。最后利用固定下载文件 `./FILE` 建立回显，既避免依赖外部连接，也能稳定判断利用是否成功。
