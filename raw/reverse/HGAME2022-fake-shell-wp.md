# fake shell

## 题目简述

主函数使用 RC4 校验输入，看似只需拿固定密钥解密目标字节，但直接照着主函数还原会得到乱码。真正的密钥在 `main()` 之前被 `__attribute__((constructor))` 标记的初始化函数替换；该函数还读取 `/proc/self/status` 的 `TracerPid`，只在未被调试时执行替换。

## 解题过程

RC4 的加密与解密是同一过程，因此首先需要确认密钥。沿 `__libc_start_main` 的 `init` 参数或 `.init_array` 交叉引用，可以找到构造函数。其核心逻辑等价于：

```c
char status[104];
int fd = open("/proc/self/status", 0);
read(fd, status, sizeof(status));

char *line = strstr(status, "TracerPid:");
if (line != NULL && atoi(line + 11) == 0) {
    strcpy(key, "w0wy0ugot1t");
}
```

`TracerPid` 为 `0` 表示进程未被调试，此时运行前密钥被改成 `w0wy0ugot1t`。如果只在调试器里观察主函数，分支状态可能与正常执行不同，这正是“看见 RC4 却解不对”的原因。

目标密文共有 32 字节：

```text
b6 94 fa 8f 3d 5f b2 e0 ea 0f d2 66 98 6c 9d e7
1b 08 40 71 c5 be 6f 6d 7c 7b 09 8d a8 bd f3 f6
```

用纯 Python 实现 RC4，可避免依赖额外密码库：

```python
def rc4(data: bytes, key: bytes) -> bytes:
    state = list(range(256))
    j = 0

    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) & 0xff
        state[i], state[j] = state[j], state[i]

    i = 0
    j = 0
    result = bytearray()
    for value in data:
        i = (i + 1) & 0xff
        j = (j + state[i]) & 0xff
        state[i], state[j] = state[j], state[i]
        stream = state[(state[i] + state[j]) & 0xff]
        result.append(value ^ stream)
    return bytes(result)

target = bytes.fromhex(
    "b694fa8f3d5fb2e0ea0fd266986c9de7"
    "1b084071c5be6f6d7c7b098da8bdf3f6"
)

print(rc4(target, b"w0wy0ugot1t").decode())
```

输出为：

```text
hgame{s0meth1ng_run_bef0r_m4in?}
```

## 方法总结

逆向 ELF 时不能把 `main()` 当作程序的绝对起点。`.init_array`、构造函数、TLS 回调和运行时加载钩子都可能提前修改密钥或状态。遇到“算法识别正确但结果始终错误”时，应检查进入主函数前的数据写入，并留意反调试逻辑是否让调试执行与正常执行走了不同分支。
