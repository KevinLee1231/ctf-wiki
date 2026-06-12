# liproll

## 题目简述

本题是 Linux kernel pwn，启用了 FG-KASLR，函数级随机化使传统基于 `vmlinux` 静态偏移的 ROP 构造变困难。漏洞位于 `cast_a_spell`：用户传入长度 `v1` 大于 `0x100` 时，`copy_from_user` 会把超过栈上 `buf[256]` 的数据写入栈帧，覆盖局部变量 `ptr` 和 `size`。

题目附件包含内核模块、启动环境和 exp，核心交互是通过 ioctl/读写路径操作全局 buffer。覆盖后的 `ptr/size` 又会被保存到 `global_buffer`，后续 read/write 依赖它们，因此可以把栈溢出转成内核任意读写，再泄露代码段定位 gadget 或直接搜索/修改 cred。

## 解题过程

题目 exp 位于 [`UESuperGate/D3CTF-2021-Exploits`](https://github.com/UESuperGate/D3CTF-2021-Exploits)。下文保留了该 exp 依赖的关键 primitive：FG-KASLR 下的内核任意读写、代码段泄露/搜索和 cred 或 ROP 提权路线。

题目灵感来源于 hxpctf 2020 kernel-rop

采用了细粒度的 kaslr ,使得 ROP 链构造时出现了困难。

FG-KASLR 开启后会导致 `vmlinux` 和相应内核模块以函数为单位分段，并在原先地址随机化的基础上打乱函数加载顺序，无法通过静态分析确定函数相对 `base_addr` 的偏移。这个机制可参考 LWN 对 [fine-grained KASLR](https://lwn.net/Articles/824307/) 的介绍。

漏洞点出在 cast_a_spell 函数中:

```c
void cast_a_spell(__int64 *a1) {
    unsigned int v1;
    int v2;
    __int64 v3;
    _BYTE buf[256];
    void *ptr;
    int size;

    if (global_buffer) {
        ptr = global_buffer;
        v1 = *((_DWORD *)a1 + 2);
        v2 = 256;
        v3 = *a1;
        if (v1 <= 0x100) {
            v2 = *((_DWORD *)a1 + 2);
        }
        size = v2;
        if (!copy_from_user(buf, v3, v1)) {
            memcpy(global_buffer, buf, *((unsigned int *)a1 + 2));
            global_buffer = ptr;
            *((_DWORD *)&global_buffer + 2) = size;
        }
    }
}
```

`v1` 是用户传入的长度参数。如果 `v1 > 0x100`，`copy_from_user` 会造成栈溢出，覆盖 `ptr` 和 `size` 两个局部变量；而 read/write 函数后续都依赖保存到 `global_buffer` 中的 `ptr` 和 `size`，因此可以把这次覆盖转化为任意读写机会。

预期解是利用题目中存在的内存泄露读取代码段，恢复需要的 gadgets，然后用栈溢出构造 ROP。另一条路线是通过任意读写暴力搜索堆空间，直接修改 `cred` 结构体。

题目 exp 的关键是先利用栈溢出改写后续读写使用的 `ptr/size`，再把它稳定转化为任意内核读写；FG-KASLR 下需要通过泄露或扫描定位仍可用的 gadget/cred。

## 方法总结

- 核心技巧：内核栈溢出覆盖后续读写所用的指针和长度，把局部变量覆盖转成任意读写 primitive。
- 识别信号：`copy_from_user(buf, user_ptr, user_len)` 的长度来自用户，长度限制只影响保存的 `size`，没有限制实际 copy 长度；后续全局状态又复用被覆盖的局部变量。
- 复用要点：FG-KASLR 下不能只靠静态偏移；先用任意读泄露代码段恢复 gadget，或用任意写搜索并改 `cred`，再完成提权。

