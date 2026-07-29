# Safenote Pro

## 题目简述

程序提供 Note 的增、删、查、改、排序和设置 Session ID 六项功能，并用自定义 `libtiny.so` 替换了 `malloc`、`free` 和输出函数。二进制启用了 PIE、Full RELRO、Canary 与 NX；启动时还通过 seccomp 禁用 `execve`、`execveat`，把标准输出复制到随机文件描述符后关闭 fd 1。

每个 Note 的结构可还原为：

```c
struct Namelist {
    char *name;
    uint32_t size;
    char *fmt;
    void (*show_func)(
        uint8_t, uint8_t, uint8_t, char *,
        uint8_t, uint8_t, uint8_t
    );
    float *magic_num;
};
```

结构体大小为 `0x28`。`name`、格式化后的 `fmt` 和 `magic_num` 分别位于独立堆块中，`show_func` 根据格式串是否含 `%s` 指向两个受限输出函数之一。利用主线是：浮点异常值破坏排序与索引计算，得到堆指针数组附近的单字节越界写；再结合自定义分配器的错位释放，逐步构造地址泄漏、任意读写和控制流劫持。

## 解题过程

### 用 inf 除以 inf 构造 NaN

浮点输入由 `strtof` 解析，而且第一个有效字符必须是数字，因此不能直接输入 `nan`。不过超大指数形式仍会被接受，例如：

```text
10e2222222222222222
```

其结果为正无穷 `inf`。排序功能在满足 $a\times b\ne 0$ 且 $c=0$ 时执行：

```c
c = a / b;
qsort(notes, 16, sizeof(void *), compar);
```

令 $a=b=\mathrm{inf}$ 即可得到：

$$
c=\frac{\mathrm{inf}}{\mathrm{inf}}=\mathrm{NaN}
$$

比较器的核心逻辑是：

```c
if (x == y)
    return 0;
if (x <= y)
    return -1;
return 1;
```

只要一侧是 `NaN`，`==` 和 `<=` 都为假，比较器便固定返回 `1`，不再满足排序所需的反对称性。`qsort` 结束后，数组首尾不再可信。

### 把 Session ID 变成可控单字节越界写

设置 Session ID 时，程序取排序后首尾元素作为最小值和最大值，并计算：

$$
i=\operatorname{int}\left(31\cdot
\frac{x-\mathrm{min}}{\mathrm{max}-\mathrm{min}}\right)
$$

随后执行：

```c
session_id[i] = *(uint8_t *)note->magic_num;
```

一旦伪造的首尾值让 `min`、`max` 不再是实际边界，索引 $i$ 就可以越过 32 字节的 Session ID。写入值又恰好是所选浮点数 IEEE 754 表示的最低字节。官方 `brute.cpp` 穷举 32 位浮点位模式，要求：

```c
(bits & 0xff) == wanted_byte
(int)(num * 31) == wanted_offset
```

于是每次“编辑 magic number → 设置 Session ID”都能在目标偏移写入一个指定字节。先用它改写 Note 指针表，再把数组布置成：

```text
0, ..., 0, NaN, num, NaN, 1, ..., 1
```

被两个 `NaN` 夹住的 `num` 可以反复修改，而不会让程序撤销“已排序”状态。

### 利用自定义 slab 的错位释放

`libtiny.so` 把堆划为若干 `64 KiB` slab，每个 slab 对应一种 $8,16,32,\ldots$ 的 slot 大小。首次分配时，分配器把堆地址高 32 位写到固定映射 `0x114514000`；后续读写、释放只重点检查：

1. 指针高 32 位是否等于记录值；
2. 指针是否落在某个 slab 的起止范围内；
3. freelist 是否有环、计数是否一致。

关键缺陷是 `free` 没有验证指针必须对齐到真实 slot 边界。通过越界改写 Note 指针，可以让删除操作释放 slab 内的错位地址，使前一个 Note 的 `magic_num` 与后一个 Note 的 `name` 重叠。后者释放后，其 freelist 后继指针会落入前者的 `magic_num`，`show` 又把该 32 位内容作为整数输出，因此先泄漏堆地址低 32 位。

接着伪造两级 `magic_num` 指针，利用形如：

```text
A -> B -> C -> D
```

的 freelist 链：一个 Note 修改中间指针的低 32 位，另一个 Note 对新地址读写。题目把合法堆、ELF 地址限制在相同的高 32 位范围内，这仍足以读取堆地址高位、程序内函数指针以及 `getrandom@GOT`，依次恢复完整 heap、PIE 和 libc 基址。

### 劫持 show_func 并执行 ORW

获得读写能力后，覆盖某个 Note 的 `show_func`。官方预期链选择 libc 中：

```asm
mov rdi, xdrs
call rcx
...
call __GI_xdrrec_endofrecord
```

把 `rcx` 设为 `gets`，第一次间接调用会把后续伪造结构读入可写堆区。控制 `xdrs` 内部字段后，`xdrrec_endofrecord` 最终执行受控函数指针；令该指针指向：

```asm
mov rsp, rdx
ret
```

即可把栈迁移到堆上的 ROP 链。

seccomp 禁止执行新程序，所以 ROP 顺序为：

1. `open("/flag", 0)`；
2. 从返回的 flag 文件描述符读取内容到堆；
3. 读取程序保存的随机输出 fd；
4. `write(random_fd, heap_buffer, 0x100)`。

本地部署中的 `flag{just_a_test_flag}` 只是占位值，真实比赛 flag 需要对远程实例执行上述利用后取得。

## 方法总结

本题的核心不是常规 glibc 堆利用，而是三层机制的组合：IEEE 754 的 `inf/NaN` 破坏 `qsort` 与归一化索引，自定义 slab 的边界检查遗漏制造错位释放，最后借结构体函数指针和 `xdrrec_endofrecord` 完成栈迁移。随机输出 fd 与 seccomp 又决定了收尾必须是读取 flag 后写回正确 fd，而不是 `system("/bin/sh")`。

出题人的 [SafeNotePro 仓库](https://github.com/TLD1027/SafeNotePro) 保留了完整 `exp.py`、浮点位模式爆破器与两种语言的题解；正文已概括利用所需的关键数据结构、漏洞原语和控制流链。
