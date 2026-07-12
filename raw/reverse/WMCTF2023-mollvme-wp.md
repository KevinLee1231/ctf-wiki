# mollvme

## 题目简述


原始方向为 `Blockchain`；归档时按主要障碍归入 `reverse`，因为解法主体是 Move 字节码反汇编、死代码消除和符号执行。

本题要求你分析 Move Bytecode。随着 Sui 和 Aptos 的发展，Move 语言逐渐流行起来。然而，目前用于 Move 分析的工具很少。

服务端会生成或提供一段 Move 字节码的 hex，参赛者需要反汇编字节码，理解其中对输入的约束，并提交一个 32 字节的 bytearray 使所有有效路径约束成立。题目仓库中的 `mutator` 目录说明了这类约束由模板和条件生成器拼接而来，因此直接靠人工读完整字节码成本较高。

## 解题过程

当你连接到挑战服务器时，它会以 hex 形式提供字节码。你应该首先使用 Move 的 Rust crate 来反汇编该字节码。以下是一个简单的代码片段：

```rust
use std::fs;

use move_binary_format::file_format::CompiledModule;

fn main() {
    // argument is file path
    let path = std::env::args()
        .nth(1)
        .expect("Expected path to bytecode file");

    // read bytecode from file
    let module_bytecode = fs::read(path).expect("Unable to read bytecode file");

    // println!("{:?}", module_bytecode);
    let compiled_module = CompiledModule::deserialize(&module_bytecode).unwrap();

    for func_def in compiled_module.function_defs {
        let code = func_def.code.as_ref().unwrap().code.clone();
        let mut i = 0;
        for bytecode in code {
            println!("{:?}: {:?}", i, bytecode);
            i += 1;
        }
        return; // we break early because the first function is the one we want
    }
}
```
阅读完易于人类阅读的反汇编字节码后，目标很明确：输入一个 32 字节的 bytearray 以满足所有约束条件。

如果仔细观察，你会注意到某些字节码永远不会被执行到。这是由于区块链语言的特性决定的：它们很少优化代码，因为 a) 这可能会违背程序员的意图 b) 这可能会引入严重漏洞。

有两条预期解法方向。首先是使用静态分析，消除死代码。存在大量类似 `if (ALWAYS_TRUE && ALWAYS_FALSE) return TRUE` 的模式。因此这是一种可行的方法。

另一个方向是使用符号执行。我只使用了大约 40 种类型的字节码，其中许多是相似的。因此实现一个执行引擎的工作量是可行的。

我写了一个非常丑的 symengine，我不打算公开它。欢迎随时私信我 (@publicqi)。

整体结构类似于 angr。有一个 `Simgr` 类，它包含一个 `State` 列表。在循环中，对所有 states 调用 `step`，并检查是否有任何 states 到达了 target。对于 `State` 的 `step` 函数，它会根据执行状态再次返回一个 `State` 列表。例如，如果某条指令的一个步骤会分叉为两条路径，它可能会返回两个 `State`。

在赛后问卷中有人说这是一个 RE 挑战，我部分同意这个说法。但我们在区块链世界确实需要更多工具，不是吗？

## 方法总结

- 核心技巧：把 Move 字节码题转化为约束求解问题，先反汇编，再消除死代码或实现轻量符号执行。
- 识别信号：输入固定长度、字节码中存在大量恒真/恒假分支和不可达路径时，应优先清理路径，而不是逐条手算所有分支。
- 复用要点：区块链字节码不一定经过传统编译器优化；题目中的无效分支可能是故意制造的噪声，符号执行时要能维护多个 `State` 并在到达目标条件时取出输入模型。
