# MmapHeap

## 题目简述

程序使用自定义分配器，在 `mmap` 得到的区域上维护链式 heap node。写入会自动补 NUL，形成 off-by-null，可破坏相邻 node 并构造重叠分配。该 mmap 区域与共享库和动态加载器 `ld.so` 保持稳定相对距离，因此重叠堆最终可以把一次分配导向 `link_map`。

配套 `libmylib` 提供真正打开文件的 `r_open` 和诱饵 `f_open`。预期目标不是传统 GOT 覆盖，而是修改 `link_map.l_addr` 的最低字节，让延迟绑定 `_dl_fixup` 把 `r_open` 的解析结果写入相邻的 `f_open@GOT`，从而令后续 `f_open("/flag")` 实际执行 `r_open`。

仓库只保留官方脚本，没有本题源码、ELF、`libmylib.so` 或 ld。完整 RELRO、canary、NX、PIE 无法确认；能够确认的是目标必须保留 lazy binding，且脚本中的 ld/link_map 偏移严格依赖部署版本。

## 解题过程

### off-by-null 构造重叠 node

官方脚本先按特定大小布置多个 node：

```python
add(0, 0x100 - 0x30 - 0x10 - 0x10, b"asd")
add(1, 0x100 - 0x20, p64(0x30000000) + p64(0))
add(2, 0x10, b"asd")
add(3, 0xfe00 - 0x10, b"asd")
add(4, 0xff90, b"asd")
free(2)
add(5, 0x20, b"a" * 0x20)
```

索引 5 的数据恰好填满用户区，输入函数追加的 `\0` 落入下一 node 的元数据。索引 1 中预置的伪大小/链字段与前后的大尺寸分配共同让分配器接受被破坏的 node，形成与后续 mmap 区域重叠的逻辑块。官方 WP 没有给出 node 结构源码，因此不能可靠为 `0x30/0x10` 等字段命名；脚本能证明这些尺寸是构造重叠的必要布局常量。

在获得重叠后，脚本用一个超大申请推进分配位置：

```python
LD_OFFSET = 0x1cef0 - 0x6000
add(6, LD_OFFSET + 0x392e0 - 0x10, b"asd")
```

这些偏移把下一次可控分配导向当前 ld 版本中的目标 `link_map` 区域。

### `_dl_fixup` 为什么受 `l_addr` 控制

运行时解析未绑定 PLT 符号时，加载器调用 `_dl_fixup(struct link_map *l, reloc_arg)`。与本题直接相关的计算是：

```c
reloc = D_PTR(l, l_info[DT_JMPREL]) + reloc_offset(..., reloc_arg);
rel_addr = l->l_addr + reloc->r_offset;
```

`reloc->r_offset` 只记录相对 ELF 基址的 GOT 偏移，最终写入地址由 `l_addr` 加上该偏移得到。ELF 映像页对齐，所以正常 `l_addr` 的最低 12 位为零；如果能把最低字节从 `0x00` 改成 `0x08`，本次解析结果就会比正常 GOT 槽向后偏移 8 字节。

这一原理来自题目引用的 [Nightmare: One Byte to ROP](https://hackmd.io/@pepsipu/ry-SK44pt)：重点不是照搬其完整攻击，而是利用 `_dl_fixup` 以 `l_addr + r_offset` 计算 resolution address 的事实。

### 把 `r_open` 解析结果写进 `f_open@GOT`

`libmylib` 中 `r_open` 与 `f_open` 的 GOT 槽相差 8 字节。首先调用一次 `load(0, "/flag")`，让路径检查使用的 `strstr` 完成正常解析，避免稍后 `l_addr` 污染影响这个前置符号。

随后让下一次分配落到 `l_addr`，写入单字节 `0x08`：

```python
add(7, 0x300, b"\x08")
```

当程序第一次解析 `r_open` 时，加载器仍能找到正确的 `r_open` 地址，但 resolution address 被整体加 8，结果落入相邻的 `f_open@GOT`。于是后续调用 `f_open` 实际跳到 `r_open`：

```text
正常：rel_addr = l_addr + r_open.r_offset      → r_open@GOT
污染：rel_addr = l_addr + 8 + r_open.r_offset  → f_open@GOT
```

官方交互顺序为：

```python
load(0, b"/flag")  # 先解析 strstr
add(7, 0x300, b"\x08")
load(0, b"asd")    # 触发 r_open 的首次解析并错位写 GOT
load(0, b"/flag")  # f_open@GOT 现已指向 r_open
show(0)
```

输入自动追加 NUL 且目标地址受 mmap 相对布局的低位影响，官方脚本以断线重连方式重试，成功率约为 $1/16$。这不是完整 ASLR 基址爆破，而是对目标低半字节布局的有限重试。

官方材料没有保存最终 flag，也没有保留可用于当前环境复跑的 ld/libmylib，因此本文无法动态验证 `LD_OFFSET`、`0x392e0` 和 GOT 间距；这些数值应视为官方部署版本的证据，而不是通用常量。

## 方法总结

- 核心技巧：通过 mmap 自定义堆的 off-by-null 获得 ld 相对地址写，再污染 `link_map.l_addr`，把 lazy binding 的 GOT 写入位置平移 8 字节。
- 识别信号：大块 mmap 分配与 ld/libc 相邻、可控 node 元数据、未解析 PLT 符号和相邻 GOT 槽组合出现时，应检查 `link_map` 与 `_dl_fixup` 的 resolution address。
- 复用要点：必须先解析不希望受污染的前置符号；`l_addr`、`DT_JMPREL`、GOT 间距和 ld 相对偏移都依赖具体 ELF/加载器，跨版本复用前必须重新测量。
