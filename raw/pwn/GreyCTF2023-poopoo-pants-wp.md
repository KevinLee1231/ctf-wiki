# GreyCTF 2023 Poopoo Pants

## 题目简述

题目是 glibc 堆利用。隐藏菜单项提供越界对象访问，并允许修改 `stdout` 的 `_IO_FILE` 字段；普通菜单中的负下标和对象生命周期错误可制造重复释放。利用链先泄露 libc、栈和堆，再绕过 safe-linking 污染 tcache，把后续分配重定向到返回地址并写入 ROP。

## 解题过程

调用隐藏选项 `1337` 并传入索引 `-4`，响应中的对象数据包含 `_IO_2_1_stdout_` 附近指针。官方脚本用泄露的第 33 到 40 字节减去 `stdout` 符号及结构内偏移，求出 libc 基址。

随后保留原 FILE 头部，只修改关键标志和读区间指针，将 `_IO_read_ptr/_IO_read_end` 指向 `environ` 附近。下一次输出会把该内存区当作待发送数据，泄露环境指针所在的栈地址；同一段输出末尾还给出堆指针。按官方实例修正固定结构偏移后，得到目标返回地址和堆基址。

利用普通菜单分配两个小块，再通过 `scoop`、`eat` 和负下标形成 tcache 重复释放。glibc safe-linking 保存的 next 指针为

$P_{enc}=P_{target}\oplus(P_{chunk}\mathbin{\gg}12)$。

因此伪造链指针时写入：

```python
encoded = stack_target ^ ((heap_base + 32) >> 12)
```

再次分配后，tcache 返回栈上的目标位置。最后一个分配把对齐用 `ret`、`pop rdi; ret`、`/bin/sh` 地址和 `system` 写成 ROP 链；函数返回即获得 shell。读取部署 flag 得到：

```text
grey{elma_p00poOoo3d_h1s_p4n7s_p0000pOooopoooop0000poooop000pOoooOp000000}
```

## 方法总结

现代堆题通常需要先完成多域泄露，再考虑 tcache 污染。这里 FILE 结构把 libc 泄露扩展为对 `environ` 的任意读，负索引和重复释放提供写原语；有了堆地址后，safe-linking 只是可逆异或编码。每个固定偏移都应以随题 libc 和当前二进制为准，不能照搬到其他版本。
