# GlacierCTF2023 - Glacier Rating

## 题目简述

C++ 程序允许创建、删除和显示 Rating，并把当前用户对象放在堆上；管理功能只在 `User.perms == 0` 时输出 flag。删除 Rating 后，保存它的指针没有清零，显示和再次删除都不会检查对象是否仍存活，形成 UAF 与 double free。

## 解题过程

先创建、删除并显示一个 Rating。glibc tcache 把空闲 chunk 的 `fd` 以 safe-linking 形式写入用户数据；当链尾为 NULL 时，泄漏值等于 `chunk_address >> 12`，乘以 `0x1000` 即可恢复堆页基址。

glibc 2.38 会检测 tcache 内的直接 double free，而程序最多只有少量可控 Rating。菜单中的 `scream into the mountains` 会把最多 50 个 `std::string` 放进 vector；长度超过 15 字节会绕过 SSO，在堆上分配。输入 7 个 16 字节字符串后函数返回，恰好填满目标尺寸的 tcache bin。

此时释放 Rating A、B、A，chunk 因 tcache 已满进入 fastbin，可形成 A-B-A 链。再分配七个对象清空 tcache，使 fastbin 中的重复 chunk 被搬回可控分配路径。把 A 的 next 指针改为用户权限字段时要按 safe-linking 编码：

$$
\text{encoded}=\text{target}\oplus(\text{chunk}\gg12).
$$

```python
perm_address = heap_base + user_perm_offset
encoded = perm_address ^ (poisoned_chunk >> 12)
add_rating(p64(encoded))
add_rating(b"padding")
add_rating(b"consume-duplicate")
add_rating(p64(0))
```

最后一次分配落到 `User.perms`，写入零后选择管理功能，得到：

```text
gctf{I_th0ght_1_c0uld_n0t_m3ss_4nyth1ng_up}
```

## 方法总结

C++ 容器和字符串最终仍由堆分配器管理，不能因为源码没有显式 `malloc/free` 就忽略生命周期漏洞。利用现代 glibc 时应同时处理 tcache 数量限制、fastbin double free 条件和 safe-linking 指针编码。
