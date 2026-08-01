# GlacierCTF 2024 static-notes

## 题目简述

服务让选手上传 `notes.json`，随后用 Zig 在服务器上重新编译并运行一个静态笔记程序。JSON 在 comptime 被解析，替换变量 `@name@` 也会被展开成一段编译期生成的内联汇编。官方 WP 为空，但源码、最终 payload 和 exploit 足以还原预期利用。

运行时先把标题复制进 0x100 字节栈缓冲区，再调用生成的 `substitute()`。初始复制可以保持在边界内，但替换辅助函数 `subcpy()` 不知道剩余空间，会从靠近缓冲区末尾的标记位置写入数百字节替换值，造成可重复的栈溢出。

## 解题过程

### 1. 控制编译期生成的替换代码

程序用 `@embedFile("notes.json")` 嵌入用户 JSON，并在 comptime 收集所有 `@...@` 标记。对每个标记，编译器生成类似：

```asm
lea sub_NAME, %rsi
call getSubstituteVariableOffset
...
lea sub_NAME_with, %rsi
call subcpy

sub_NAME:      .asciz "@NAME@"
sub_NAME_with: .asciz "ATTACKER_VALUE"
```

这意味着替换值不仅参与数据处理，还作为 `.asciz` 字节串嵌入最终 ELF 的固定代码区域。官方 payload 借此放入两类材料：

- 替换 `A`：由短指令组成的滑道和最终 `execve` syscall stub；
- 替换 `B`：填充后跟 `/bin/sh` 字符串。

构建配置为 `ReleaseFast`、strip，官方生成物没有阻止该控制流覆盖的栈 canary，且相关嵌入地址在同一构建环境中稳定。

### 2. 从合法长度标题触发替换溢出

`list()` 对每条 note 执行：

```zig
var title_buffer: [0x100]u8 = std.mem.zeroes([0x100]u8);
substitute(&title_buffer, note.title);
```

包装函数先做：

```zig
@memcpy(buf[0..content.len], content);
Notes.substitute(&buf[0]);
```

官方 payload 让原始标题长度不超过 255，例如用 252 个 `B` 后接 `@1@`，因此第一次 `memcpy` 不越界。真正的问题在：

```zig
export fn subcpy(buf: [*:0]u8, sub: [*:0]u8) void {
    const sub_len = std.mem.indexOfSentinel(u8, 0, sub);
    for (0..sub_len) |i| buf[i] = sub[i];
    buf[sub_len] = 0;
}
```

替换位置已经接近 `title_buffer` 末尾，而 `@1@` 对应的替换值约 600 字节；`subcpy` 没有目标长度，因而越过缓冲区、保存寄存器和返回地址。

### 3. 用多条 note 拼出含零字节的返回状态

一次长替换会在末尾自动写 NUL，也容易破坏尚未布置的栈字节。官方 `payload.json` 因此使用 28 条 note 和 `A、B、1、C、S、2、3、N、4` 共 9 个替换项：

1. 第一条 dummy note 只为让生成器把 shellcode 和 `/bin/sh` 字串嵌入 ELF；
2. 后续标题把替换标记依次放在偏移 252、251、250 等位置；
3. 每次调用 `list()` 的循环体复用同一栈布局，向前移动一个字节的标记让 `subcpy` 的结尾 NUL 落在所需位置；
4. 特殊替换项再写入 syscall 号 59、`/bin/sh` 地址、零参数和最终代码地址。

这种“由多次溢出逐字节修补”的方法解决了 JSON/comptime 字符串中嵌入多个 NUL 以及单次覆盖会提前截断的问题。最终覆盖让 `list()` 返回到替换 `A` 中的指令序列；该序列通过若干 `pop` 恢复：

$$
\mathrm{rax}=59,\quad
\mathrm{rdi}=\&\texttt{"/bin/sh"},\quad
\mathrm{rsi}=0,\quad
\mathrm{rdx}=0,
$$

随后执行 `syscall`，即 `execve("/bin/sh", NULL, NULL)`。

### 4. 触发 list 并读取 flag

上传只包含 `payload.json` 的 tar.gz，等待服务端 Zig 构建成功，选择不下载二进制，然后向启动的笔记程序发送：

```text
list
```

循环处理到最后一组恶意标题时返回链生效并弹出 shell。执行 `cat /flag.txt` 得到：

```text
gctf{WHY_W41T_F0R_RUNT1M3_BUG2_WH3N_Y0U_H4V3_C0MPT1M3?}
```

## 方法总结

本题的“静态”并不意味着安全：用户数据在编译期决定汇编和常量布局，运行时替换器又把无长度指针当作无限缓冲区。利用时应分别控制原始标题长度和替换后长度，先让初始复制安全通过，再从末尾标记触发溢出；多次调用及 NUL 终止副作用还能变成精确写栈工具。修复应给替换函数显式传入容量、拒绝扩展后超长内容，并避免用不可信编译期数据生成内联汇编。
