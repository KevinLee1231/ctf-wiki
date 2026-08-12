# dbench_jumbf

## 题目简述

这是一个运行在 `pwn.red/jail` 中的 C2PA/JUMBF JPEG 解析服务。它循环接收十进制 `jpeg size` 和等长十六进制 JPEG，提取 APP11 段中的 JUMBF，再打印描述框和 JSON/XML 前 256 字节。目标二进制使用 Debian Trixie 的 glibc；flag 位于 `/flag.txt`。

JUMBF 的外层 `Lbox` 声称重组后框的总长度。解析器据此分配缓冲区，却没有保证 APP11 段实际拼接的字节总数不超过它；后续又把框中不可信的长度用于打印。于是同一接口同时提供了未初始化堆数据泄露和跨 chunk 写入。

## 解题过程

### 关键观察

`db_extract_jumbfs_from_jpg1` 遇到一个新的 JUMBF 实例时按 `Lbox` 分配；随后直接按 APP11 计算出的 payload 长度复制。后续同 `En`、递增 `Z` 的段继续向移动后的 `jumb_buf` 复制，缺少剩余空间检查。

```cpp
jumb_buf = new unsigned char[box_length];   // box_length 来自 Lbox
memcpy(jumb_buf, data,
       static_cast<size_t>(this_app11_paylaod_size) + this_box_header_size);
jumb_buf += this_app11_paylaod_size + this_box_header_size;

// 后续分段
memcpy(jumb_buf, data, this_app11_paylaod_size);
jumb_buf += this_app11_paylaod_size;
```

此外，`DbBox::deserialize` 在 `lbox == 0` 时把框长当作调用者提供的剩余长度，`print_box` 会将 JSON/XML payload 写到标准输出。这让带有短实际内容、但留下堆尾部的 JUMBF 读出已经释放的堆数据。

### 泄露 libc 与 safe-linking key

官方 solver 先发送约 `0x500` 字节 JPEG，令其释放的 chunk 落入 unsorted bin。第二个 JPEG 声明 `Lbox=0x200`，但只放入最小 JUMD/JSON 结构；服务打印 `Data` 时读到尾部残留。泄露中偏移 9 起的六字节是 unsorted-bin 指针，据此计算：

```python
libc_base = int.from_bytes(leaked[9:15], "little") - 0x1e5b20
```

用 `Lbox=0x400` 重复这一方法可得到空 tcache chunk 中残留的编码 fd，即本页的 safe-linking key。官方布局中目标 `T` chunk 在下一页，因此使用 `t_key = heap_key + 1`。这些偏移只适用于题目固定的 Trixie glibc。

### 溢出、tcache 投毒与 FSOP

随后构造三个 JUMBF 实例来布置相邻 chunk：额外的 `T`、大小为 `0xf8` 的 `S`、以及请求大小 `0x268` 的 `T`。攻击 JPEG 的第一个 APP11 段令 `S` 的声明大小较小而实际复制更长，从而覆盖相邻 `T` 的 tcache fd；写入值是

```python
poisoned_fd = (stdout_addr - 0x10) ^ t_key
```

第二个框取走一次正常 `T`，第三个框则从被投毒的 tcache 得到 `stdout-0x10`，并把伪造的 `_IO_FILE_plus`、`_IO_wide_data` 和 wide vtable 写入标准输出对象。关键字段是：

```python
fsop[0x00:0x04] = b"  sh"             # system 的字符串参数
w64(0x28, 1)                           # 进入 overflow 路径
w64(0x88, stdout_addr + 0x150)         # _lock
w64(0xa0, stdout_addr + 0xe0)          # _wide_data
w64(0xd8, libc_base + 0x1e4228)        # _IO_wfile_jumps
w64(0x1c8 + 0x68, libc_base + 0x53110) # fake wide-vtable 的 __doallocate=system
```

`_IO_wfile_jumps` 是合法的 libc vtable，因而能通过 vtable 范围检查；宽字符缓冲分配最终调用 `system("  sh")`。取得 shell 后发送 `cat /flag.txt` 即可。官方 solver 的四阶段顺序是“libc 泄露 → heap key 泄露 → chunk 布局 → tcache/FSOP”，不能把它们合并或颠倒，否则堆状态和 safe-linking 编码会失效。

成功输出为 `grey{jumb0_0v3rfl0w_1n_4_jumbf_b0x_6imryubogc09}`。

## 方法总结

- 核心技巧：文件格式重组时，声明长度与累计复制长度不一致，先泄露堆，再做 tcache poisoning 与 `stdout` FSOP。
- 识别信号：解析器以不可信 length 分配、跨多个 segment 复制却不维护剩余大小；格式打印函数又会读取未初始化或越界 payload。
- 复用要点：先区分“分配不足造成的写”与“长度解释造成的读”，再按 glibc 版本计算 libc/safe-linking/FSOP 偏移。FSOP 对 glibc 内部字段极其敏感，固定运行时是利用前提而不是可忽略的部署细节。
