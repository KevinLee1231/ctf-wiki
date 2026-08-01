# GlacierCTF 2024 Schrödinger Compiler

## 题目简述

服务接收 Base64 编码的 `.tar.gz`，从中取出 C++ 源码并执行 `g++ main.cpp`，但丢弃全部编译输出，也不会运行生成的程序。单连接的总时限为 3 秒。

`/flag.txt` 的内容本身带双引号，可以通过预处理器 `#include` 直接初始化 `constexpr std::string_view`。随后用 `if constexpr` 比较指定位置和候选字符：猜对时快速编译结束，猜错时强制编译器计算极其昂贵的模板递归并触发超时，由此建立逐字符 timing oracle。

## 解题过程

### 1. 在编译期读入 flag

仓库中的 flag 文件格式是：

```cpp
"gctf{...}"
```

所以源码可以写成：

```cpp
#include <string>

static constexpr std::string_view flag =
#include "/flag.txt"
;
```

预处理后它等价于一个正常字符串字面量，不需要运行可执行文件，也不依赖编译错误回显。

### 2. 构造编译耗时分支

定义按字符和位置实例化的模板：

```cpp
template <char CHR, char POS>
constexpr void check() {
    if constexpr (CHR == flag[POS]) {
        static_assert(true);
    } else {
        ct_factorial<800>();
    }
}
```

`ct_factorial<N>()` 在每一层重复请求大量 `N * ct_factorial<N-1>()` 子表达式，错误分支使 constexpr 求值和模板实例化成本远超 3 秒。正确分支在编译期被裁剪，服务会很快输出固定的 `Bye` 行。

### 3. 逐位置枚举字符

对位置 $i$ 和候选字符 $c$ 生成：

```cpp
int main() {
    check<static_cast<char>(CANDIDATE), POSITION>();
}
```

官方 exploit 的候选集为大小写字母、数字、下划线和花括号。每次连接提交一个源码包：

- 在约 1 秒内收到 `Bye`，说明 `c == flag[i]`；
- 连接超时或没有结束标记，说明进入了昂贵分支。

脚本并发测试同一位置的候选，找到字符后继续下一位，直至 `}`。需要重试超时边界上的结果，并用完整 `gctf{...}` 格式校验最终串。恢复得到：

```text
gctf{1420_1S_Th3_C0mP1ler_D34D_0R_4l1v3_2928}
```

出题人的[技术说明](https://ecomaikgolf.com/posts/glacierctf2024-schrodinger-compiler/)包含完整模板和并发 exploit；本文已转写编译期 include、分支放大器和 oracle 判定规则。

## 方法总结

本题把编译器本身变成了侧信道执行环境：即使没有诊断输出和运行阶段，预处理、常量求值及资源消耗仍能泄漏秘密。在线编译服务不能把秘密挂载进同一命名空间；还应同时限制 CPU、内存、模板深度和并发，并避免仅用“是否超时”作为可远程区分的响应状态。
