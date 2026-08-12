# Amnesia

## 题目简述

题目要求提交一个 32 位 C 程序，输出 `Hello, world!` 并正常退出。第一关在编译后清零 ELF 的 `.data` 与 `.rodata`；第二关清零 `.text`。核心是理解编译器把指令、常量、全局数据和函数放入哪些 section，并有意识地把必要内容放到不会被清除的位置。

## 解题过程

### 轻度失忆：避开 data 与 rodata

字符串字面量通常位于 `.rodata`，所以：

```c
printf("Hello, world!\n");
```

在清零后只会看到空字符串。单个字符常量则可作为立即数编码进 `.text`，逐字调用 `putchar` 即可：

```c
#include <stdio.h>

int main(void) {
    putchar('H'); putchar('e'); putchar('l'); putchar('l');
    putchar('o'); putchar(','); putchar(' '); putchar('w');
    putchar('o'); putchar('r'); putchar('l'); putchar('d');
    putchar('!'); putchar('\n');
    return 0;
}
```

这里需要的是每个字符的立即数，而不是指向只读字符串的指针。

### 记忆清除：把代码移出 text

GCC 的 `section` 属性可把函数放到自定义 `.text2`：

```c
extern int payload(void) __attribute__((section(".text2")));
```

目标 GCC/链接器把 `.text2` 紧接在 `.text` 之后，并给它可执行权限。清零 `.text` 后，CPU 在题目环境中仍能沿零字节区域执行到后面的 `.text2`。需要注意：`00 00` 在 x86 上实际是 `add byte ptr [eax], al`，并不是严格意义的 NOP；本解依赖题目给定编译器、初始寄存器与映射布局使这段“滑行”没有提前故障。

普通 C 函数序言还可能调用位于 `.text` 的 `__x86.get_pc_thunk.bx`。使用 `-O` 配合内联汇编可以避免该调用，并按目标构建的取指对齐补一个零字节：

```c
#include <stdio.h>
#include <unistd.h>

char format[] = "Hello, world!\n";

int payload(void) {
    asm(
        ".byte 0;"
        "movl $format, (%esp);"
        "call printf;"
        "movl $0, (%esp);"
        "call fflush;"
        "movl $0, (%esp);"
        "call _exit;"
    );
}

extern int payload(void) __attribute__((section(".text2")));

int main(void) {}
```

字符串位于未被第二关清理的 `.data`，有效指令位于 `.text2`，库调用经过仍可用的 PLT。`fflush` 很重要：判题通过重定向捕获 stdout，直接 `_exit` 前若不刷新，缓冲内容可能丢失。

官方还记录了一个非预期方向：让清零步骤因资源不足而失败，判题脚本又未检查返回码，于是原始 ELF 被直接执行。它属于 checker 错误，不如 section 解法稳定，也不应作为理解题目机制的主线。

## 方法总结

- 核心技巧：根据 ELF section 的清除范围重新布置常量和代码，第一关使用立即数字符，第二关使用可执行的自定义 section。
- 识别信号：判题在编译后按 section 名修改 ELF，而不是验证程序源代码意图。
- 复用要点：section 名、排列、权限、函数序言和指令对齐都依赖工具链；必须用题目指定的 GCC 与链接环境检查 `readelf`、`objdump` 结果，不能把零字节误当作通用 NOP sled。
