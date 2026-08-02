# TSGCTF2020 Detective WP

## 题目简述

每次连接先提交一个 flag 下标，服务只把对应的一个字节保存到全局变量。菜单允许申请、释放两个堆指针，并提供一次任意正向偏移单字节写：

```c
void read_flag(void) {
    unsigned idx = get_index();
    unsigned offset = get_num();
    ptrs[idx][offset] = flag;
    flag = 0;
}
```

程序没有 show 功能，无法直接读回该字节；但 flag 中间 32 字符已被 `sanity_check` 限定为 `[0-9a-f]`。目标是把未知字节写进 fastbin 的 `fd` 最低字节，再用 glibc 对伪造 chunk 的 size 校验构造“连接存活/进程 abort”的字符判定 oracle。

## 解题过程

题目使用 glibc 2.31。`calloc` 在该版本的路径中不直接从 tcache 返回块，因此先对目标大小连续申请、释放 7 次填满 tcache，后续同尺寸释放块进入 fastbin。通过两个可保存指针和若干垫块，整理出近似关系：

```text
fastbin: C -> B

A: 重新取回后作为 read_flag 的基准块
C: fd 最低字节将被 flag[i] 覆盖
target: 与 B 仅最低地址字节不同，候选位置放有合法 size
```

ASLR 不影响同一堆页内的最低字节。假设当前候选为 `c`，先在偏移与 `ord(c)` 对应的位置写入目标 fastbin 大小的伪造头，例如用户大小 `0x20` 对应 chunk size `0x31`。数字 `0x30` 至 `0x39` 和字母 `0x61` 至 `0x66` 落在同一页的不同区间，官方脚本分别用两种垫块布局把 `0x31` 放到候选地址前 8 字节。

随后取回 A，并让一次性写入越过 A，落到 C 的 `fd` 低字节：

```python
alloc(0, 0x10, b"B")
read_flag(0, 0x1a0)
```

写入后，C 的下一节点地址最低字节恰好等于真实 `flag[i]`。再连续从相同 fastbin 申请：

- 若猜测正确，`fd` 指向预先放好 `0x31` size 的候选位置，glibc 接受伪造 chunk，第二次 `calloc` 正常返回；
- 若猜测错误，`fd` 落到没有合法对齐 size 的位置，fastbin 校验失败并调用 `abort`，远端连接断开。

因此每个字符最多建立 16 次连接：

```python
known = "TSGCTF{"
for index in range(7, 39):
    for candidate in "0123456789abcdef":
        io = connect()
        select_flag_index(io, index)
        prepare_fastbin_and_fake_size(io, candidate)
        write_flag_byte_into_fd(io)
        try:
            consume_poisoned_fastbin(io)
        except EOFError:
            io.close()
            continue
        known += candidate
        io.close()
        break
known += "}"
```

逐位恢复结果为：

```text
TSGCTF{67f7d58ac9301f273d16aec9829847b0}
```

## 方法总结

本题把一个“只能写、不能读”的秘密字节转成堆分配器错误 oracle。关键是先利用已知字符集缩小候选，再让秘密控制 fastbin `fd` 的低字节，并为某个候选地址单独准备合法 chunk 头。进程是否通过 allocator 完整性检查就等价于布尔查询。保护秘密时不能认为单向写原语天然无泄漏；只要秘密会影响崩溃、延迟或资源分配结果，攻击者就可能把它变成逐位侧信道。
