# only

## 题目简述

题目是一个带 seccomp 的 amd64 菜单程序，包含 notes 和 bookkeeping 两组功能。seccomp 只禁止 x32 ABI 系统调用以及 `execve`、`execveat`，因此 `read`、`open`、`sendfile`、`personality` 等仍可使用。

程序有两个相互配合的漏洞：bookkeeping 把最多 36 个 `double` 写入只有 32 项的栈数组，能够越过 canary 写到保存的 `rbp` 和返回地址；输入 IEEE-754 位模式 `0x0d0e0a0d0b0e0e0f` 还会进入一次性隐藏菜单，在 RWX 匿名映射中执行最多 9 字节的用户代码。总 PDF 用隐藏菜单直接搭两阶段 shellcode，题目目录的预期解则先调用 `personality`，再把堆变成可执行区并利用栈溢出跳转。

## 解题过程

### 1. 用浮点输入触发隐藏菜单

`bookkeeping` 以 `%lf` 读入数值，却又按整数比较其原始 64 位表示：

```c
if (*(size_t *)&tokens[i] == 0x0D0E0A0D0B0E0E0F)
    backdoor();
```

把该位模式按 IEEE-754 double 解释，保留 17 位有效数字后可发送：

```text
8.5925645443139351e-246
```

隐藏菜单的选项 1 申请一页 `PROT_READ | PROT_WRITE | PROT_EXEC` 内存，先写入一段清零通用寄存器并临时平移 `rsp`、`rbp` 的固定 stub，再在偏移 56 处读取最多 9 字节，最后追加恢复栈帧和 `ret` 的尾部。因此这 9 字节必须完成第一阶段目标，且可以依赖进入用户片段前 `rax`、`rdi` 等已被清零。

### 2. 总 PDF 的直接两阶段路线

总 PDF 发送的第一阶段为：

```asm
pop rdx
pop rsi
pop rdx
pop rsi
pop rsi
pop rsi
syscall
```

机器码仅 8 字节。`rax = 0`、`rdi = 0` 已由前置 stub 保证，若当前构建的调用现场在平移后的栈上提供了合适值，若干 `pop` 会令 `rsi` 指向可执行映射、`rdx` 足够大，最终执行 `read(0, rsi, rdx)`。随后发送 `0x100` 字节 NOP sled 和 `open("/flag")`、`read`、`write` 的 ORW shellcode即可输出 flag。

这条路线很短，但 PDF 没有推导被弹出的栈值，因而明显依赖附件二进制与运行时栈布局；迁移或复现失败时，应优先采用下述单题 WP 的预期路线。

### 3. 预期解：9 字节设置 `READ_IMPLIES_EXEC`

容器以 `seccomp=unconfined` 启动并由程序自行安装 BPF 过滤器，所以未被 BPF 禁止的 `personality` 可直接调用。9 字节恰好能执行：

```asm
mov edi, 0x440000
mov al, 0x87
syscall
```

系统调用号 `0x87` 是 `personality`。参数 `0x440000` 同时含 `ADDR_NO_RANDOMIZE` 和 `READ_IMPLIES_EXEC`；后者使后续只请求读权限的映射也获得执行权限，并影响 brk 管理的堆页。固定 stub 会在用户代码后恢复 `rsp`、`rbp`，所以这条第一阶段不需要碰栈即可安全返回菜单。

### 4. 泄露堆基址并把短代码写入堆

notes 的 `edit` 条件写反：只有 `ctx.ptr == NULL` 时才调用 `read(0, NULL, 0)`，实际不能编辑已分配块。仍可借 `save` 泄露和写入：

1. 申请并释放一个 `0x10` 块，再申请相同大小；未初始化内容保留了 tcache safe-linking 的编码 next。
2. `save("log.txt")` 在成功路径用 `%s` 打印 `ctx.ptr`，泄露低 5 字节。单元素 tcache 的编码值等于 `chunk_addr >> 12`，左移 12 即可得到堆页基址。
3. 通过不同尺寸的申请/释放消耗 tcache，再连续申请 25 个 `0x1000` 块，使 brk 扩展到可预测位置。由于已经设置 `READ_IMPLIES_EXEC`，这些堆页可执行。
4. `save` 的文件名最多读 12 字节。传入“两字节填充 + 9 字节读入 stub + `/`”，以目录结尾的路径令 `open(..., O_CREAT | O_RDWR)` 失败。错误路径把文件名写入栈上 `buf`，随后 `perror` 为拼接错误信息在堆上产生一份含相同字节的临时字符串，于是短代码落入可执行堆。

写入堆的 9 字节第一阶段为：

```asm
xor edi, edi      /* fd = 0 */
mov rsi, rbp      /* 第二阶段目的地址 */
mov dl, 0x40
syscall           /* read(0, rbp, 0x40) */
```

### 5. 越过 canary 并执行 ORW

`bookkeeping` 的循环条件是 `i < 36`，会覆盖 `tokens[32..35]`。在附件编译布局中，依次对应局部变量/填充、canary、保存的 `rbp` 和返回地址。利用流程为：

1. 先输入 33 个普通非零 double，推进到 canary 槽。
2. 在 canary 位置输入单独的 `+`。`scanf("%lf", ...)` 转换失败，目标内存保持原值，但程序忽略返回值并继续递增 `i`，从而不改写 canary。
3. 把“堆上第一阶段地址 + 9”按 double 位模式输入为保存的 `rbp`，把第一阶段起始地址输入为返回地址。
4. `bookkeeping` 返回后跳到堆上短代码；它把第二阶段读到 `rbp` 指向的紧随区域，系统调用结束后顺序落入新写入的 ORW shellcode。

seccomp 禁止 `execve`，所以第二阶段应使用 `open("flag")` 配合 `sendfile`，或 `open/read/write`。仓库容器中的 flag 为：

```text
RCTF{f878b22c-5542-4b51-9780-d1188750f3ce}
```

## 方法总结

- 隐藏入口要求按位构造 double，不能把十六进制整数直接作为十进制数发送；稳定做法是用 `struct.pack/unpack` 转成 17 位 double 字符串。
- 总 PDF 的 8 字节 `read` stub 最短，但依赖平移后栈上的具体值；预期解通过 `personality(READ_IMPLIES_EXEC)`、tcache 泄露和 `perror` 堆拷贝建立更可解释的可执行堆。
- `%lf` 转换失败而程序忽略返回值，可用来“跳过” canary 槽；这要求确认失败输入会被消费，并在目标 libc/附件上实测。
- 最终必须遵守 seccomp：不要尝试 `execve`，应以 ORW 或 `sendfile` 读取 flag。
