# DownUnderCTF 2020 - formatting

## 题目简述

附件是一个小型 ELF。程序会打印诱饵字符串 `DUCTF{haha its not that easy}` 的主体，却在未打印的栈缓冲区中用 `sprintf` 拼出真正的 flag。核心是追踪格式化函数的数据流，而不是相信标准输出中的第一段 `DUCTF{...}`。

## 解题过程

反编译后可以看到关键变量与调用：

```c
char *flag = "DUCTF{haha its not that easy}";
char *fmt = "%s%02x%02x%02x%02x%02x%02x%02x%02x}";

int crap = 0x6f;
int this = 0x29;
int is   = 0xaa;
int too  = 0x29;
int easy = 0x90;
int what = 0xac;
int the  = 0xbc;
int heck = 0x36;

sprintf(realflag + 6, fmt, "d1d_You_Just_ltrace_",
        this, crap, is, too, easy, what, the, heck);
puts(flag + 6);
```

`puts(flag + 6)` 输出的是诱饵字符串去掉 `DUCTF{` 后的部分；真正的结果被写进 `realflag`。格式串要求八个整数分别按两位小写十六进制输出，参数顺序是：

```text
29 6f aa 29 90 ac bc 36
```

加上程序先写入的 `DUCTF{`、固定文本和右花括号即可得到：

```text
DUCTF{d1d_You_Just_ltrace_296faa2990acbc36}
```

动态分析同样很短。`ltrace` 能显示 `sprintf` 的实参或结果；也可以在 `sprintf` 返回后用 GDB 查看目的缓冲区。以源码为依据的静态拼接则不需要执行附件。

## 方法总结

分析“stack strings”或运行时拼接字符串时，应沿 `sprintf`、`snprintf`、`memcpy` 等写入函数追踪目的缓冲区，而不是只搜索明文或观察最终打印内容。格式说明符和参数顺序共同决定结果，本题尤其要保留 `%02x` 产生的两位十六进制表示。
