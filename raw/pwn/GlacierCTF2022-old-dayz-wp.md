# GlacierCTF2022 - Old dayz

## 题目简述

题目运行在旧版 glibc 上，提供 note 的分配、释放、写入和查看功能。`delete` 只调用 `free(notes[i])`，不清空指针和长度，所以同一索引在释放后仍可读写；利用目标是先从 unsorted bin 泄漏 libc，再做 fastbin poisoning 覆盖 `__malloc_hook`。

## 解题过程

先分配一个用户大小 `0x80` 的 chunk，再在其后放一个小 chunk 防止它与 top chunk 合并。释放前者后，其用户区开头被写入指向 `main_arena` 的 unsorted-bin 指针，而 `view(0)` 仍通过悬空指针按 `%s` 输出内容：

```python
add(0, 0x80)
add(1, 0x10)
delete(0)
leak = view(0)
libc_base = leak - 0x3c4b78
```

重新申请 `0x80` 清理 unsorted bin。随后分配并释放一个用户大小 `0x60` 的 fastbin chunk；`write` 同样没有生命周期检查，可在释放后覆盖其 fd。将 fd 改到 `__malloc_hook` 前方一个具有合法 `0x7f` size 字节的位置：

```python
target = libc_base + 0x3c4b10 - 0x23
write(10, p64(target))
add(11, 0x60)
add(12, 0x60)
```

第二次分配返回 hook 附近。填充 19 字节后把 one-gadget 地址写到 `__malloc_hook`。最后再次进入 `add`；官方脚本把 0x100 个零字节作为 size 输入，使 `atoi` 得到 0，同时把调用栈上 one-gadget 要求为空的区域清零。随后 `malloc(0)` 触发 hook，取得 shell 并读取：

```text
glacierctf{pwn_1S_Th3_0nly_r3al_c4t3G0ry_4nyw4y}
```

## 方法总结

旧版 glibc 没有 tcache 与 safe-linking，使经典的“unsorted bin 泄漏 + fastbin dup/poisoning + malloc hook”链条可以直接使用。根因仍是对象释放后未撤销所有访问路径；清空指针、记录生命周期并拒绝对已释放索引执行 view/write 才能消除原语。
