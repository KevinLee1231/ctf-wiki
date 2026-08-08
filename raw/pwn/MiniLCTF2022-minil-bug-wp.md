# MiniLCTF2022 minil_bug Writeup

## 题目简述

程序读取最多 512 字节整数型 VM 字节码并交给一个简易虚拟机执行。补丁只禁止 `LOAD/GLOAD` 的负索引，却没有保护操作数栈指针。VM 对象在堆上的布局为指针字段、`stack[]`、`call_stack[]`；通过栈下溢可以读写 `code` 与 `globals` 指针，再把 `globals` 指向 `main` 的栈上 `code` 数组，实现对主线程栈的 32 位粒度任意读写。

## 解题过程

关键结构可抽象为：

```text
VM + 0x00  code pointer
VM + 0x08  code_size
VM + 0x10  globals pointer
VM + 0x18  nglobals
VM + ...   stack[0]
           call_stack[]
```

相对于 `stack[0]`，`stack[-8:-7]` 是 `code` 指针，`stack[-4:-3]` 是 `globals` 指针。先执行一次 `CALL` 令 `callsp=0`，获得合法局部变量区；再用 `POP` 让 `sp` 向负方向移动，把原 `globals` 与 `code` 指针高低 32 位暂存进 locals。随后把 `globals` 改为 `code`，`GLOAD/GSTORE` 就变成对 `main` 栈数组的读写。

已知比赛构建中 `code[134:136]` 对应 `main` 保存的返回地址。读取其低 32 位，减去附件 libc 2.31 的 `__libc_start_main_ret` 偏移 `0x240b3`，得到 libc 基址的低位；高位保持不变。然后由 VM 自身的 `ICONST/IADD/ISUB/LOAD/STORE` 计算并写入以下 ROP：

```text
ret
pop rdi ; ret
address of "/bin/sh"
system
```

生成字节码的核心辅助函数如下：

```python
def real_addr(offset):
    return LOAD + p32(1) + LOAD + p32(0) + ICONST + p32(offset) + IADD

def write_global(index):
    return GSTORE + p32(index) + GSTORE + p32(index + 1)
```

写完 ROP 后必须恢复原 `globals` 指针，否则 `vm_free()` 会对栈地址执行 `free()`。最后发送 `HALT` 并让 `main` 返回，控制流进入已写好的 libc ROP。源码中的读取循环还存在 `code+nread` 按 `int *` 缩放的偏移错误，可用于分段越界写；上述 VM 下溢路线只需一次完整 512 字节输入，更容易稳定复现。

## 方法总结

补上负索引检查并不等于修复 VM：栈指针、全局区上界和调用栈都必须分别验证。利用的核心是先把一个受限数组读写原语升级成指针重定向，再借 `globals` 访问宿主栈。构造过程中地址被拆成两个 32 位整数，且所有 libc 运算都在 VM 内完成；恢复被劫持的堆指针是避免利用成功前崩溃的必要收尾。
