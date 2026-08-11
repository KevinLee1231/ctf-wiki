# DownUnderCTF 2023 roppenheimer Writeup

## 题目简述

程序把最多 32 个原子保存到 `std::unordered_map<unsigned int,uint64_t>`。发射中子时，它把目标 bucket 的全部元素复制到只有 19 个元素的栈数组，却没有限制 bucket 大小。只要构造 32 个始终落入同一 bucket 的键，就能用键值对覆盖返回地址。

## 解题过程

libstdc++ 在本题插入数量下依次使用 1、13、29、59 个 bucket，而整数的默认哈希基本就是自身。离线生成键时，要求它们与目标 `0x13371337` 在这些模数下同余：

```cpp
if (target % 13 == value % 13 &&
    target % 29 == value % 29 &&
    target % 59 == value % 59) {
    collisions.push_back(value);
}
```

加入目标和 31 个碰撞键后，目标 bucket 含 32 项。`copy(atoms.begin(bucket), atoms.end(bucket), elems)` 把 32 个 `pair<unsigned int,uint64_t>` 写入 `elems[19]`，其键、对齐填充和 64 位 data 都由攻击者预测或控制。

程序特意提供了 `pop rax; pop rsp; pop rdi; ...; ret` 组合 gadget。通过给两个特定碰撞键设置 data，将保存返回地址覆盖为该 gadget，并把新的 `rsp` 指向全局 `username` 缓冲区。第一阶段先把 ROP 写进用户名：

```python
stage1 = flat(
    b"A" * 16,
    pop_rdi,
    elf.got.puts,
    0,                  # gadget 额外 pop rbp
    elf.plt.puts,
    elf.sym.main,
)
```

栈迁移后调用 `puts(puts@got)` 泄漏 libc，再回到 `main`。此时执行栈仍位于全局用户名区域，第二次 `fgets(username, ...)` 会覆盖当前伪栈；根据泄漏计算 libc 基址后，写入 `execve("/bin/sh",0,0)` ROP 链即可直接接管控制流。

```text
DUCTF{wH0_KnEw_Th4T_HAsHm4ps_4nD_nUCle4r_Fi5S10n_HAd_s0meTHiNg_1n_c0MmoN}
```

## 方法总结

哈希表平均分散并不是容量保证，攻击者可主动制造碰撞。把 bucket 内容复制到定长栈数组前必须以实际容量为界。本题还展示了 C++ pair 布局如何把碰撞键的 data 字段变成 ROP 数据，以及全局输入缓冲区如何充当稳定的栈迁移目标。
