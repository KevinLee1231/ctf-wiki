# Truth

## 题目简述

本题给出源码和编译环境，表面上对 `data` 到 `backup` 的长度做了检查，但源码中存在未定义行为：返回路径附近有空指针引用。Clang 某些优化 pass 会基于 UB 折叠控制流和检查逻辑，导致源码看似有保护，实际编译产物中出现堆溢出。

题目附件包含 C++ 工程源码和编译产物，核心陷阱是源码中看似存在的检查会因未定义行为和优化被折叠。利用主线是先通过实际调试确认编译后 check 被优化掉，再用堆溢出覆盖下一个 node 的虚函数指针，结合 heap/libc 泄露构造 one_gadget 或堆上 ROP。

## 解题过程

题目源码和 exp 见 [`ZhouZiY/hctf2021_pwn`](https://github.com/ZhouZiY/hctf2021_pwn)。下文保留了仓库中真正影响利用的部分：源码检查与优化后产物不一致、UB 触发 `SimplifyCFG` 折叠、堆溢出后的虚表劫持。

题目给了源码和编译环境。源码表面上对从 `data` 到 `backup` 的拷贝长度做了检查，直接审计时看不到明显越界；但返回路径附近存在空指针引用，这是后续优化折叠的触发点。

这道题的灵感来自 BlackHat Europe 2020 议题《Finding Bugs Compiler Knows But Does Not Tell You: Dissecting Undefined Behavior Optimizations In LLVM》：<http://i.blackhat.com/eu-20/Wednesday/eu-20-Wu-Finding-Bugs-Compiler-Knows-But-Does-Not-Tell-You-Dissecting-Undefined-Behavior-Optimizations-In-LLVM.pdf>。

Clang 的某些 pass 在发现 UB 后会基于“UB 不应发生”的假设优化控制流。这里检测到空指针引用后，优化器折叠了上面的 check，导致编译产物中后续堆溢出变得可达。

确认堆溢出可达后，利用路径就比较直接：泄露 heap/libc 地址，溢出覆盖下一个 node 的虚函数指针。

后续可以构造 XML 布置 one_gadget 约束，也可以找 gadget chain 在堆上做 ROP。关键验证点不是源码表面逻辑，而是优化后汇编中 check 是否真的被去掉。

本题相关的控制流折叠主要来自 `SimplifyCFG`。`-O2` 下也可能触发类似行为，因此复现时要以题目给出的编译环境和实际产物为准。

如果对 UB 带来的问题有兴趣，还可以读 John Regehr 对 [undefined behavior and compiler optimization](https://blog.regehr.org/archives/213) 的说明。

附件工程的关键信息是对象链表/节点结构和被优化掉的 check：源码审计只能发现风险，最终必须以优化后汇编和运行时行为确认漏洞是否可达。

## 方法总结

- 核心技巧：利用编译器对 UB 的优化副作用，区分“源码里看似存在的检查”和“编译后真实执行的检查”。
- 识别信号：源码中出现空指针引用等 UB，且关键边界检查与 UB 路径在同一控制流区域；不同优化级别或 pass 下行为可能变化。
- 复用要点：遇到给源码和编译环境的 pwn 题，不能只审源码语义；要结合实际编译产物/调试结果确认检查是否被优化折叠，尤其关注 Clang `SimplifyCFG` 这类控制流优化。

