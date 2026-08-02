# N1CTF 2021 - easyX11

## 题目简述

服务端启动一个 X11 客户端程序，并把该客户端与 X Server 之间的原始 TCP 数据转发给选手。选手可以把连接桥接到自己的 X Server，再让本地经过补丁的 Fcitx5 充当 XIM 输入法服务器，向远端程序注入任意二进制“文本”。

漏洞位于题目程序的 UTF-8 转 Latin-1 函数：目标是栈上 10 字节数组，却没有目标长度参数。官方利用先借 `strlen/XDrawString` 越界读取泄露 libc，再覆盖返回地址构造 ROP。

## 解题过程

### 把远端 X11 客户端接入本地显示

服务端 `run.py` 在本机随机监听一个端口，把 `DISPLAY` 设置为该端口减 6000 后启动 `./x11`，并将建立的 X11 连接双向转发到题目标准输入输出。因此选手并不是连一个图形界面，而是临时扮演 X Server。

官方 `exp.py` 一端连接题目服务，另一端连接本地 Unix X Socket，然后双向转发：

```python
remote_xclient = remote(HOST, PORT)
local_xserver = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
local_xserver.connect("/tmp/.X11-unix/X0")

# 两个线程分别转发 remote_xclient -> local_xserver 和反方向
```

这样远端创建的 `N1CTF Easy X11` 窗口会出现在本地桌面，并可通过本地 XIM 服务向它输入内容。

### 利用无边界的 UTF-8 转换

程序在 `main` 中只有：

```c
char overflow[10] = "Easy X11";
mainLogic(overflow);
```

输入缓冲区 `buff` 会根据 `XBufferOverflow` 正确扩容，但接下来调用的转换函数没有接收目标容量：

```c
void utf8ToLatin(unsigned char *dst, unsigned char *src, size_t len) {
    int dst_idx = 0;
    for (size_t src_idx = 0; src_idx < len; ) {
        if (src[src_idx] <= 0x7f) {
            dst[dst_idx++] = src[src_idx++];
        } else if (src[src_idx] == 0xc2) {
            dst[dst_idx++] = src[src_idx + 1];
            src_idx += 2;
        } else if (src[src_idx] == 0xc3) {
            dst[dst_idx++] = src[src_idx + 1] + 0x40;
            src_idx += 2;
        }
    }
}
```

把任意 Latin-1 字节先执行 `payload.decode('latin-1').encode('utf-8')`，该函数就会在远端还原原字节，并持续覆盖 10 字节栈数组之后的内容。

官方给 Fcitx5 Quick Phrase 模块打补丁：从 `/tmp/exp` 读入任意二进制字符串，并把关键字 `exp` 映射到该字符串。每次改写 payload 后重新加载短语，在窗口中输入 `exp` 即可触发。

### 第一阶段泄露 libc

第一段内容为：

```python
leak = b"plusls".ljust(0x12, b"a")
```

转换函数写满 18 字节后没有为 `overflow` 添加终止 NUL。随后 `refresh()` 调用 `strlen(overflow)` 与 `XDrawString`，于是把栈上的后续指针也作为文本送入 X11 流。转发线程在数据中定位 `plusls` 前缀并读取其后的 6 字节：

```python
leak_addr = u64(packet[pos + 18:pos + 24] + b"\x00\x00")
libc_base = leak_addr - 0x27e4a
```

官方脚本还用泄露地址低 12 位是否为 `0xe4a` 进行版本/偏移校验。

### 第二阶段 ROP 取得 shell

第二段 payload 以 `1919810` 开头并填充到 18 字节，然后覆盖保存的返回地址。程序检测到该前缀后会从 `mainLogic` 返回，正好触发 ROP：

```text
dup2(3, 0)
dup2(3, 1)
system("/bin/sh")
```

文件描述符 3 是题目 X11 代理连接；把它复制到标准输入输出后，shell 直接复用现有通道。官方录屏逐帧显示了泄露地址 `0x7fc62bbade4a`、`gen getshell success`、`whoami` 返回 `ctf`，以及最终执行 `cat flag` 的结果：

```text
n1ctf{now_y0u_k0ow_xim}
```

仓库 `file/flag` 中的 `n1ctf{1145141919810}` 只是本地部署占位值，与录屏中的比赛 flag 不同。录屏画面承担的只是终端文字验证，相关信息已完整转写，因此不在 WP 中保留视频帧。

## 方法总结

这道题同时考查 X11/XIM 数据流和栈溢出。安全的输入缓冲区扩容并不能弥补后续转换函数缺失目标长度；编码转换尤其要同时跟踪源长度与目标容量。利用时借 XDrawString 把栈数据带回协议流，再用同一 XIM 通道传输 UTF-8 包装的任意 ROP 字节，形成了完整的泄露与控制流劫持链。
