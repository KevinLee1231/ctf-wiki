# vtable4b

## 题目简述

这是一个 C++ 虚表（vtable）教学向利用题：

- 程序创建 `Cowsay` 对象于堆上（`new Cowsay(new char[0x18]())`）；
- 暴露 `1/2/3` 菜单，分别是：
  - `1`：调用 `cowsay->dialogue()`；
  - `2`：执行 `std::cin >> cowsay->message();`；
  - `3`：`display_heap()` 打印堆布局和相关指针。

源码（`challenge/main.cpp`）明确给出 `<win>` 地址提示，并指出可通过 vtable 调用。

`task.yml` 标注该题为 `warmup/pwn`，也说明这是针对对象模型的入门级 exploit。

## 解题过程

### 关键观察

`Cowsay` 的首个机器字是 vptr，随后才是 `message_` 指针。`option 1` 会取 vptr 指向的虚表首项并间接调用；`option 2` 则向只有 `0x18` 字节的 message 缓冲区做无界 token 输入。该缓冲区先分配，`Cowsay` 对象随后分配，赛事构建的两个小 chunk 相邻，因此从 message 越界可以覆盖对象首部的 vptr。官方脚本利用 `option 3` 泄露 message 地址，并直接在 message 内伪造一项虚表：

```python
addr_win = int(sock.recvregex("<win> = 0x([0-9a-f]+)")[0], 16)
...
sock.sendlineafter("> ", "3")
addr_message = int(sock.recvregex("0x([0-9a-f]+)")[0], 16) + 0x10
...
payload  = p64(addr_win)
payload += b"A"*0x18
payload += p64(addr_message)

sock.sendlineafter("heap\n> ", "2")
sock.sendlineafter("Message: ", payload)

sock.sendlineafter("> ", "1")
```

### 利用链

1. **泄露与定位**

   先通过菜单 `3` 触发 `display_heap`，从输出提取 message 区域地址，脚本中用 `+ 0x10` 还原最终写入起点。

2. **在 message 中放置假虚表并改写 vptr**

   - message 起始的前 8 字节放置 `win`，作为假虚表第 0 项；
   - 随后的 `0x18` 字节填满 message chunk 余下空间和两个 chunk 之间的布局区域，使总偏移达到 `0x20`；
   - 最后 8 字节覆盖相邻 `Cowsay` 对象的 vptr，令其指向 message 起始地址。这里写入的 `addr_message` 是**假虚表地址**，不是在修复 `message_` 字段。

3. **触发**

   重新执行 `option 1` 时，程序先从被改写的 vptr 取得 message 地址，再读取该地址的首个机器字作为虚函数地址，最终跳到 `win()` 并执行 `/bin/sh`。

### 影响范围与偏移

脚本层面的关键值是 `+0x10` 和从 message 到对象 vptr 的 `0x20`：`display_heap` 从 `valid_message-0x10` 的 chunk 头开始打印，所以首个泄露地址加 `0x10` 才是 message 用户区；`p64(win)+b"A"*0x18` 总长正好为 `0x20`，下一枚指针才落到对象 vptr。

进入 shell 后读取根目录中的 flag 文件，得到：

```text
CakeCTF{vt4bl3_1s_ju5t_4n_arr4y_0f_funct1on_p0int3rs}
```

## 方法总结

- 核心技巧：把 message 的无界写延伸到相邻对象 vptr，同时在 message 起点布置以 `win` 为首项的假虚表，再通过虚调用劫持控制流。
- 识别信号：出现 `virtual` 方法、可写相邻堆对象和 heap dump 地址泄露时，应优先检查 vptr 是否可达；`std::cin >> char*` 只识别 token 边界，不会自动遵守目标缓冲区长度。
- 复用要点：先区分 chunk 头、用户区和对象地址。假虚表内容与对象 vptr 是两个不同写入位置：前者保存函数地址，后者必须指向前者。
