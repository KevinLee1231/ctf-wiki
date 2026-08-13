# Its A Real Heap Challenge

## 题目简述

程序用全局 `repos[0x100]` 保存堆指针，但分配条件写成 `repo_count <= MAX_REPOS`，允许第 257 次分配越界写到紧邻数组的全局 `repo_count`。利用链通过大规模堆喷把这一字节级全局越界扩展成任意索引读写，最终在栈上布置 ROP。

## 解题过程

先反复申请接近 `0x10000` 的块，并周期性用 `calloc(-1, 1)` 触发失败分配，形成约 1 GiB 的可预测堆布局。第 257 次 `repos[repo_count++]` 会把返回指针写过数组末尾，覆盖 `repo_count`，使后续 `repo_id < repo_count` 对超大索引成立。

释放交错的大块进入 unsorted bin，随后扫描越界索引，寻找落在堆喷区域的有效指针。借 `examine_property` 泄露堆地址，再修改某个 `repos` 项，使它指向任意地址：

```text
越界索引读取堆块 -> heap base
覆写 repos 指针 -> arbitrary read/write
```

将指针指向 unsorted-bin 元数据，读取 libc 指针并计算基址；再指向 `libc.environ` 泄露栈地址。最后把目标改到保存的返回地址，在栈上写入：

```text
ret
system
"/bin/sh" address
```

触发函数返回得到 shell，读取：

```text
grey{just_kidding_there_was_no_heap_involved_solved_the_heap_challenge_without_heap_feng_shui_and_pure_spraying}
```

## 方法总结

`<=` 与 `<` 的单字符错误发生在全局指针数组边界，影响却取决于链接布局：越界槽正好覆盖计数器。由于 ASLR 只留下少量堆对齐不确定性，官方解法用 1 GiB 喷射和扫描换取稳定命中，再把大计数器转为任意索引。该利用资源消耗很大，复现前应使用题目容器并确认内存上限。
