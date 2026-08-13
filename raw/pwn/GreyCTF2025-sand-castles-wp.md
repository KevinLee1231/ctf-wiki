# Sand Castles

## 题目简述

目标是 C++ “城堡”管理程序，每座城堡包含动态 `std::vector<int>`。`gnomes` 与 `move` 操作在元素数量／容量管理上产生越界和悬空布局，官方 exploit 将其转化为 tcache poisoning、任意地址读写，最终覆盖 `stderr` 的 `FILE` 结构执行 `system`。

## 解题过程

先构造两个小 vector，释放其中一个，再向另一个追加元素。`look` 输出越界整数可拼出 64 位 heap 指针，由固定相对偏移得到 heap base。随后按大小交错申请 A–G 多个 vector 并释放部分块，让 $0x20$ 与 $0x60$ tcache 链形成可控布局。

扩展 A 时覆盖相邻已释放块的 safe-linking 指针。glibc tcache 指针编码为：

$$
encoded=(chunk\_address\gg12)\oplus target.
$$

把 target 设为另一座城堡 G 的 vector 元数据，后续分配就落到该元数据上；重写 G 的 `begin/end/capacity`，即可让 `look(G)` 读取任意地址、`gnomes(G,...)` 写任意地址。先指向 unsorted-bin 残留指针泄露 libc base。

最终把一个受控 vector 指向 `_IO_2_1_stderr_`，写入伪造的 wide `FILE`：设置 `_IO_write_ptr > _IO_write_base`，让 `_wide_data` 和 vtable 指向可控位置，并把调用链中的函数位置放成 `system`、flags 中放入 `"  sh\0"`。触发错误路径的 `exit()` 时，glibc 刷新 `stderr` 并跳入伪造 wide vtable，得到 shell：

```text
grey{y0u_br0ke_th3_c4stl3s}
```

## 方法总结

这是现代 glibc/C++ 堆利用的组合题：先用 vector 越界泄堆，再绕 safe-linking 污染 tcache，构造任意读写后泄 libc，最后使用 FSOP。各偏移强依赖题目附带的 libc、libstdc++ 与二进制，复现时必须使用同版本文件；网络速度还会影响批量整数输入，因此官方脚本将一组数字拼成一次发送。
