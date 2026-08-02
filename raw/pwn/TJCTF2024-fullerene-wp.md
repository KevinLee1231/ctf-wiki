# fullerene

## 题目简述

题目是一个 C++“球面体素网格服务”。用户可以创建、删除、读写和更新 `SphereChunk`。最终目标不是直接拿 shell，而是调用程序内置 `win()`：该函数读取全局 `file_name` 指定的文件；默认值为 `oom.txt`。程序主动泄漏 `&winptr` 和 `file_name` 地址，其中 `winptr` 保存 `win` 的函数地址。

利用链包含两个阶段：先用浮点体积计算造成的单字节堆越界构造重叠 chunk，取得对象字段读写；再用 House of Einherjar 式后向合并与 tcache poisoning，把 `file_name` 改成 `flag.txt`，最后伪造 C++ 虚表调用 `win()`。

## 解题过程

`make_SphereChunk` 以浮点数 `vol` 同时参与分配和边界判断：

```cpp
in->verts = (Voxel*)smalloc(in->vol);  // 转 int 后分配
...
if (idx < 0 || idx >= vol)
    return verts[0];
return verts[idx];
```

选择精心构造的球面参数后，`smalloc` 截断得到 $n$ 字节，而 `vol` 因浮点误差略大于 $n$。于是索引 $n$ 通过比较，却写到已分配区末尾之后一字节。官方脚本将三个 voxel 区依次布置为 `0x78`、`0x10`、`0x78`，再把中间 chunk 的 size 低字节改大。释放它并执行 `heapChunk` 后，新的 `SphereChunk` 对象与第三块 `reader` 重叠。

`SphereChunk` 中的重要偏移为：

```text
updater   +0x48
noise_gen +0x50
verts     +0x58
```

通过重叠的 `reader` 可读出堆上 `n->verts` 指针，并可覆写 `n->updater`。虚调用 `updater->update_mesh(...)` 需要两级解引用，正好利用程序泄漏的全局槽：令 `updater` 指向一个堆伪对象，伪对象首地址保存 `&winptr`；虚调用先把 `&winptr` 当成 vtable，再从中取出其保存的 `win` 地址。

在触发调用之前，还要改变 `file_name`。官方 exploit 的堆步骤是：

1. 在堆块 `A` 中布置自指的假空闲 chunk；
2. 用另一次单字节越界清除后方 `C` 的 `PREV_INUSE`，并把 `prev_size` 指回假 chunk；
3. 先填满对应 tcache，再释放 `C`，迫使 glibc 后向合并，得到与仍在使用块重叠的大 chunk；
4. 从重叠块修改一个 `0x30` tcache entry 的 `fd`。glibc safe-linking 下写入值为

   $$
   \text{encoded}=\text{target}\oplus(\text{chunk\_addr}\gg12);
   $$

5. 连续两次同尺寸分配，使第二次返回全局 `file_name`。源码在它前面放置填充数组，保证该地址低字节为零并满足对齐；向其中写入 `flag.txt\x00`。

最后发送 `mupdate`。伪造的 `Updater` 虚调用进入 `win()`，它使用已修改的文件名读取并输出：

```text
tjctf{h#3p-Lee3k-WW9279s86}
```

## 方法总结

- 漏洞根源是同一浮点量在“转整数分配”和“浮点边界比较”中的语义不一致，特制参数能制造稳定的一字节越界。
- 重叠对象既提供堆地址泄漏，也提供 C++ 对象字段写入；虚表劫持需要明确写出 `object -> vtable -> function` 两级指针关系。
- 新版 glibc 的 tcache poisoning 必须按 chunk 地址编码 safe-linking 指针。本题将后向合并、重叠 chunk、safe-linking 和虚调用串成一条链，每一步都由下一步所需的地址或写原语驱动。
