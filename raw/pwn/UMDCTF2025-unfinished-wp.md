# UMDCTF 2025 - unfinished

## 题目简述

程序看似只是一个没写完的堆题：从全局数组 `number[128]` 读取分配数量，再执行
`new int[n]`。实际存在两个相互配合的条件：

```cpp
fgets(number, 500, stdin);
long n = atol(number);
int *chunk = new int[n];
```

`fgets` 可越界覆盖 `.bss` 中后续对象；极大的 `n` 又会使 `new[]` 抛出
`std::bad_alloc`，强制进入 C++ 异常展开流程。攻击目标是覆盖 libgcc 的
`registered_frames`，伪造 CIE/FDE，并让异常展开器调用 `sigma_mode()`。

## 解题过程

### 1. 同时制造溢出和异常

payload 以一个巨大、以 NUL 结尾的十进制数开始：

```python
payload = b"999999999999999\x00"
```

NUL 保证 `atol` 只解析这段数字，后面的二进制伪造数据不会影响结果。
`new int[999999999999999]` 无法完成分配，于是抛出 `std::bad_alloc`。与此同时，
后续 499 字节输入已经越过 `number`，能够覆盖相距 336 字节的
`registered_frames`。

### 2. 在 `number` 中伪造异常帧索引

libgcc 的动态帧注册数据先通过一棵按 PC 范围索引的树查找对象，再从对象中取得
排序后的 FDE 数组。官方脚本在 `number` 内依次构造：

```text
number + 0x10  单叶节点 FDE btree
随后          registered object
number + 0x60 仅含一个 FDE 指针的数组
number + 0x98 伪造 FDE
紧随其后      伪造 CIE
```

btree 只有一个叶节点，范围从 `0` 覆盖到 `0xffffffffffff`，所以异常展开时的任意
程序计数器都能命中它。object 的 sorted 标志被置位，避免 libgcc 尝试重排伪造
数组。数组中的唯一指针指向 `number + 0x98`。

伪造 FDE 的 `pc_begin` 为 0、`pc_range` 覆盖整个地址空间，并通过负相对偏移指向
紧随其后的 CIE。CIE augmentation 中加入 `P` 标记，并把 personality routine
指针设置为二进制内的 `sigma_mode()`：

```python
payload += b"AAAAAP\x00AAAA"
payload += p64(binary.symbols["_Z10sigma_modev"])
```

最后将 payload 填充到偏移 336，把 `registered_frames` 改成伪造 btree：

```python
payload = payload.ljust(336, b"A")
payload += p64(binary.symbols["number"] + 16)
```

### 3. 让展开器主动调用目标函数

输入处理结束后，超大分配抛出 `std::bad_alloc`。展开器查询已经被替换的
`registered_frames`，伪造范围匹配当前 PC，随后解析伪造 FDE/CIE，并把
`sigma_mode()` 当作 personality routine 调用。该函数执行
`system("/bin/sh")`，最终读到：

```text
UMDCTF{crap_i_have_to_come_up_with_a_flag_too?????????}
```

## 方法总结

本题把一个 `.bss` 溢出与攻击者可控的异常触发点组合起来。普通控制流数据可能离
全局数组很远，但运行库的异常注册表同样属于高价值间接控制流元数据。审计 C++ 程序
时，超大 `new` 不只是错误处理问题；如果附近还有可覆盖的 CIE/FDE 索引，
`std::bad_alloc` 本身就可以成为触发利用链的“调用指令”。修复应同时限制输入长度、
检查分配数量并避免让可写全局区邻接运行库控制结构。
