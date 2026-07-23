# gnalekcip

## 题目简述

程序使用自定义 `Unpickler` 执行 `pickle.pkl`。`find_class` 只允许 `operator.add`、`operator.getitem`、`struct.pack/unpack` 和 `input`，但 `persistent_load` 会把 persistent ID 当成另一段 pickle 递归执行，并共享顶层 memo。生成器借此把 pickle opcode 组合成带寄存器、子程序和控制流的解释器。

仓库没有独立 solve，但公开的生成器同时保留了构造逻辑和生成时使用的正确输入。

## 解题过程

先理解两层执行模型：

- `BINPUT`、`BINGET` 等 memo opcode 充当寄存器；
- `REDUCE` 调用白名单函数；
- `BINPERSID` 触发 `persistent_load`，递归执行一段共享 memo 的 pickle，效果类似子程序调用；
- `scramble_regs()` 不断重新分配 memo 槽位，用来干扰线性反汇编。

若只有 `pickle.pkl`，可以用 `pickletools.genops` 递归解析所有 persistent ID，并记录 memo 槽位的符号值；遇到 `operator.getitem/add` 与 `struct.pack/unpack` 时执行符号化简，最后收集对输入各字符的比较约束。

在赛后公开仓库中，更直接且没有歧义的证据位于 `gen.py` 的 `main()`：生成器构造 payload 后，会用一个常量字符串调用 `run_payload` 做自测。该字符串就是能走到 success 分支的完整输入：

```text
UMDCTF{pick3l_or_elkcIP_wh1cH_0ne_1s_i7_504f4c4c5554414e54}
```

将其交给 `pickelang.py`，payload 的所有字符运算、末尾常量比较和内部数独式约束均通过。

## 方法总结

pickle 不只是序列化格式，它的栈机 opcode 足以表达计算。白名单限制了可调用函数，却没有阻止递归 persistent ID 与共享 memo 组合成完整解释器。赛后复盘时应以生成器自测输入作为最高置信度答案，同时说明只拿到 handout 时需要递归反汇编和符号执行，而不能简单对 pickle 做普通字符串搜索。
