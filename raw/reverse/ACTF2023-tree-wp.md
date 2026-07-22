# Tree

## 题目简述

附件只有一个去符号的 64 位 PIE 可执行文件 `chall`。程序接受一份“不包含头文件”的 C++ 源文件，以 `-std=c++11` 调用 Clang 前端构造抽象语法树（AST），再用两组 ASTMatcher 检查源码结构。普通程序只会得到 `nonono`，满足全部结构约束时输出 `Right!`。

题目名中的 Tree 指的是 Clang AST。官方提示还特别说明，不应直接拿完整的 `clang` 主程序做 BinDiff：主程序不一定实例化本题使用的全部 ASTMatcher 模板，符号恢复会缺项。更可靠的方法是根据反汇编中出现的节点类型和常量编写小型 ASTMatcher 参考程序，再对对应模板实例进行比对。Clang 官方文档也说明了 matcher 的组合与绑定语义：节点 matcher 可以通过 `bind("id")` 保存匹配节点，并在回调中用同一 ID 取回；这正是二进制中字符串 `1`、`2`、`f` 和 `class` 的用途。[Clang LibASTMatchers 文档](https://clang.llvm.org/docs/LibASTMatchers.html)

## 解题过程

先运行帮助信息确认输入形式：

```text
USAGE: chall [options] <source file with no header>
```

静态分析可定位到两个全局 matcher 的构造函数，以及最终的匹配回调：

- `0x513c90` 构造并绑定 ID 为 `f` 的 `forStmt` matcher；
- `0x5141d0` 构造名称为 `AAA`、绑定 ID 为 `class` 的 `cxxRecordDecl` matcher；
- `0x53e670` 根据绑定结果更新三个全局计数；
- `0x55a0b0` 仅在三个计数分别满足 `1`、`1`、`5` 时输出 `Right!`。

第一组 matcher 中能直接恢复出 `++`、`<`、整数 `10` 以及 `equalsBoundNode("1")`。因此可确定它寻找如下形式的循环：初始化部分声明循环变量，条件使用同一变量与 `10` 作小于比较，递增部分使用 `++`。每找到一个这样的循环，`f` 计数加一。

该 matcher 还会检查循环中的 `BinaryOperator`。自定义谓词读取 `BinaryOperator::getOpcode()`：操作码为 `0x19` 时计分加一，否则减一。Clang 14 中 `0x19` 对应 `BO_AddAssign`，也就是 `+=`。循环条件中的 `<` 同样是二元运算，因此每个合格循环天然贡献一次 `-1`。

最终要求 `f == 5` 且运算符计分为 `1`。构造 5 个合格循环会产生 5 个 `<`，总分为 `-5`；再在循环体内放入 6 个 `+=`，总分变为

$5\times(-1)+6\times(+1)=1$。

第二组 matcher 先筛选名为 `AAA` 的类，回调再检查继承图：

1. `AAA` 必须恰有两个直接基类；
2. 两个直接基类都必须恰有一个直接基类和一个虚基类；
3. 两条路径最终指向同一个虚基类。

这正是一个共享虚基类的菱形继承结构。组合全部条件后，可提交以下无头文件源码：

```cpp
class Base {};
class A : virtual public Base {};
class B : virtual public Base {};
class AAA : public A, public B {};

int main() {
    for (int i = 0; i < 10; ++i) {
        i += 1;
        i += 1;
    }
    for (int i = 0; i < 10; ++i) {
        i += 1;
    }
    for (int i = 0; i < 10; ++i) {
        i += 1;
    }
    for (int i = 0; i < 10; ++i) {
        i += 1;
    }
    for (int i = 0; i < 10; ++i) {
        i += 1;
    }
    return 0;
}
```

本地验证：

```text
$ ./chall answer.cpp
Right!
```

## 方法总结

本题的决定性障碍是恢复 Clang ASTMatcher，而不是普通控制流或字符串校验。分析时应从全局 matcher 构造函数中的节点类型、操作符字符串、绑定 ID 和回调逻辑入手；必要时编写最小 ASTMatcher 参考程序辅助 BinDiff。动态调试则适合直接观察三个全局计数，将复杂 matcher 拆成“循环数量、二元运算计分、类继承结构”三个可独立验证的条件。最终输入只需在 AST 层满足约束，源码本身无需执行。
