# Christmas Bash

## 题目简述

题目继续使用 Slang Christmas VM，但开放了 `open`、`write`、`memcpy` 等宿主函数，并增加反汇编功能。服务先下载选手提供的 `.scom`，单独执行一次 `-r file`；只有输出精确等于 `hello`，才在同一新进程中执行 `-r file -d file`。

利用依赖三处机制：

- 反汇编器 `line_fmt` 用 `vsprintf` 向 0x80 字节栈缓冲区格式化任意长度的单词，形成栈溢出；
- VM 的 `gift_t.item` 只是无类型的 64 位整数，同一值可被当作数值、指针或函数地址，允许指针算术和类型混淆；
- VM 内存中保留 `sleep` 函数指针，可推导对应版本 libc 的基址。

## 解题过程

### 确认反汇编栈溢出

反汇编行的格式化函数为：

```c
void line_fmt(line_t *this, int addr, char *fmt, ...) {
    char buffer[0x80];
    va_list ap;
    va_start(ap, fmt);
    vsprintf(buffer, fmt, ap);
    va_end(ap);
    /* 保存输出行 */
}
```

编译一个名字极长的 `gift`，再用 `-d` 反汇编即可覆盖返回现场。附件版本中，从长字符串起点到伪栈指针的距离为 `0x14c`：

```text
"a" * 0x14c + fake_stack_pointer
```

由于 `.scom` 中的字符串按显式长度读取，可以包含地址字节；但最初下载的文件仍需同时通过第一次运行检查，所以不能直接把它做成只会崩溃的溢出样本。

### 两阶段自修改 scom

利用程序把将要再次加载的文件本身作为可写目标。初始文件设置 `check=1`，第一分支完成两件事：

1. 输出恰好 `hello`，使 server 进入第二条命令；
2. 打开自身 `.scom`，把开头的常量索引/opcode 从“令 `check=1`”改成“令 `check=2`”。

第二条命令首先以 `-r` 重新加载修改后的文件，因而进入 `check=2` 分支。该分支再次打开自身，把文件头和 opcode 改成预先调试好的“超长 gift 名”反汇编样本，随后写入：

```text
overflow header/opcodes
0x14c bytes padding
pointer to heap fake stack
```

同一个进程接着处理 `-d file`，重新加载已经变形的 `.scom`。反汇编长名字时触发 `vsprintf` 溢出，返回地址/栈指针落到堆上的伪栈。

### 在 VM 堆上构造 ROP

`gift_t` 的关键定义是：

```c
typedef struct gift {
    char *name;
    uint64_t item;
} gift_t;
```

`item` 没有类型标签，Slang 的加减运算可把字符串指针、`sleep` 指针和普通整数混用。利用文件中已有字符串取得一块稳定堆地址，再由 `sleep - sleep_offset` 求 libc 基址，计算当前 libc 的 `system`、`pop rdi; ret` 和对齐 `ret`。用 `Vixen` 对应的 `memcpy` 逐项写入伪栈：

```text
pop rdi; ret
address_of("cat /flag")
ret
system
```

这些偏移与附件 libc 严格绑定；官方远端示例使用的 `sleep`、`system` 和 gadget 偏移只适用于当时镜像，复现时应在配套 libc 中重新计算。将无副作用的 `cat /flag` 放入字符串区即可直接回显，无需沿用旧脚本中的外连命令。

完整状态变化为：

```text
初始文件(check=1)
  -> 第一次 -r：输出 hello，并改成 check=2
  -> 第二进程 -r：写出溢出版 scom，并在同进程堆上布置 ROP
  -> 同进程 -d：line_fmt 栈溢出，迁移到堆 ROP
  -> system("cat /flag")
```

仓库没有保存部署时的 flag 文件或远程最终回包，因此本篇不补造 flag 字符串；源码、官方利用逻辑和成功条件已经足以复现完整链。

## 方法总结

本题的难点不是单独的 `vsprintf`，而是如何跨过服务端的两次运行并把不可直接输入的地址送到溢出现场。自修改 `.scom` 让第一次运行只负责通过 `hello` 检查，第二次 `-r` 再把文件改成攻击形态，紧接的 `-d` 才触发溢出；无类型 `gift.item` 与 `memcpy` 则负责在同一进程堆上生成 ROP。复现时应分别验证三道边界：首次输出必须完全等于 `hello`，第二次文件写入偏移必须与加载格式一致，ROP 偏移必须来自配套 libc。
