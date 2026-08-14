# CakeCTF2021 UAF4b

## 题目简述

程序在堆上分配一个 16 字节的 `COWSAY` 结构，其中包含函数指针和消息指针：

```c
typedef struct {
    void (*fn_dialogue)(char *);
    char *message;
} COWSAY;
```

删除操作只调用 `free(cowsay)`，却没有清空全局指针。之后“修改消息”仍会通过这个悬空指针写字段，形成典型 Use-after-Free。程序还直接泄露了 `system` 地址，使函数指针劫持非常直接。

## 解题过程

### 让新分配复用已释放结构

先执行一次 Delete，16 字节结构对应的堆块进入 tcache。Change 操作会执行 `malloc(17)`；在 64 位 glibc 中，这个请求与原结构落入同一个 0x20 大小类，因此第一次 Change 很可能取回刚释放的块。

当输入为 `p64(system)` 时，新块内容覆盖悬空结构起始处的函数指针：

```python
free_cowsay()
change_message(p64(leaked_system))
```

### 设置函数参数并触发调用

第一次 Change 结束时，原 `cowsay->message` 字段已经被写成刚复用块的地址。第二次 Change 再分配一个消息块并写入 `/bin/sh`，同时通过悬空结构更新第二个字段：

```python
change_message(b"/bin/sh")
use_cowsay()
```

Use 操作原本执行：

```c
cowsay->fn_dialogue(cowsay->message);
```

现在等价于 `system("/bin/sh")`。取得 shell 后可读取：

```text
CakeCTF{U_pwn3d_full_pr0t3ct10n_b1n4ry!N0w_u_kn0w_h0w_d4ng3r0us_UAF_1s!_ea2e5f3e}
```

这条链依赖附件 glibc 的分配大小类和 tcache 复用顺序，但题目刻意让结构和 17 字节消息请求落入同一大小类。

## 方法总结

- `free` 后不清空长期保存的指针，会让后续同大小分配把新对象内容解释成旧结构。
- UAF 利用首先应比较旧对象大小与可控分配大小，确认能否落入同一 allocator bin。
- 函数指针与其参数都位于悬空结构中时，可以分两次分配分别控制，形成清晰的调用劫持。
