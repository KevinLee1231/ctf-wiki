# vector overflow

## 题目简述

程序以 `std::cin >> buf` 把未限长的文本写入全局 `char buf[16]`。其后把全局 `std::vector<char> v` 当作需要等于 `DUCTF` 的五字节序列；满足后调用 `win()` 并执行 `/bin/sh`。因此关键不是覆盖返回地址，而是越过 `buf` 覆盖相邻 `std::vector` 的三个内部指针，伪造一个长度为 5 的可控字符区间。

## 解题过程

### 关键观察

libstdc++ 的普通 `std::vector` 对象保存 `begin`、`end`、`capacity_end` 三个指针。源码中 `buf` 只有 16 字节，而 `v` 是紧随其后的全局对象；官方 solver 将输入排成：16 字节填充、三个 64 位指针、8 字节填充和字符串 `DUCTF`。

它令：

```python
payload = (
    b"x" * 16
    + p64(buf + 0x30)      # begin
    + p64(buf + 0x35)      # end，begin + 5
    + p64(buf + 0x35)      # capacity_end
    + b"x" * 8
    + b"DUCTF"
)
```

这样 `v.size()` 计算为 $5$，范围 for 循环会从伪造 `begin` 读取五个字节，恰好逐个匹配栈上的 `ductf = "DUCTF"`。它不需要修改函数指针或返回地址。

### 验证链

循环通过后，程序直接执行 `win()`，其源码为 `system("/bin/sh")`。官方 solver 以目标 ELF 的 `buf` 符号为基址构造上述 payload；题目配置给出的验证 flag 是 `DUCTF{y0u_pwn3d_th4t_vect0r!!}`。本文未运行 shell 或 exploit，只转写静态 payload 布局和源码成功路径。

## 方法总结

- 核心技巧：C++ 容器对象被相邻溢出污染后，先检查其指针不变量能否被伪造，而非一开始就寻找 ROP。
- 识别信号：固定大小全局数组、未限长 C++ 输入以及相邻 `std::vector`，常意味着可控制迭代范围、`size()` 或元素来源。
- 复用要点：三个指针的偏移依赖 ABI、标准库实现和二进制布局；本题的 `buf + 0x30/0x35` 来自随附目标，不应照搬到不同构建。
