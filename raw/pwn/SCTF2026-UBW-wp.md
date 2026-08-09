# UBW

## 题目简述

程序是带 `add/free/show/merge` 的菜单堆题。真正漏洞在 `merge(dst, src)`：当 `dst == src` 且 `realloc` 搬迁原块时，旧指针先被 `realloc` 内部释放，随后又被 `strcat(new, old)` 读取并被 `free(old)` 第二次释放。通过控制第一次释放时的合并结果，同一地址可以按不同 size 进入两个 bin，形成泄漏和 tcache poisoning。

二进制还包含父进程 CFI/hash gate 与 “Unlimited Blade Works” 字符串，但它只校验若干 table-call hash，既不验证 syscall 编号，也不足以稳定设置 ORW 参数。官方解法没有走该复杂链，而是继续利用堆原语做 largebin attack 和 FSOP。

## 解题过程

### 1. 触发自合并双重释放

逻辑可抽象为：

```c
new = realloc(arr[dst], strlen(arr[dst]) + strlen(arr[src]) + 1);
strcat(new, arr[src]);
free(arr[src]);
arr[dst] = new;
arr[src] = NULL;
```

令 `dst == src`，并通过相邻块布局迫使 `realloc` 返回新地址。旧块第一次释放后进入 unsorted 并可能与相邻空闲块合并；`strcat` 随后从已释放区域读取数据，`merge` 的 “Tempered” 输出会带出 unsorted/main_arena 指针，用来计算 libc 基址。第二次 `free(old)` 依据已经改变的 chunk header 把同一地址送入另一个 tcache/fastbin size class。

### 2. 建立 tcache poisoning

先填满所需 tcache，再用自合并制造重复引用。根据 safe-linking 规则，伪造 next 指针时写入：

$$
\mathrm{encoded}=\mathrm{target}\oplus(\mathrm{chunk\_addr}\gg12).
$$

官方 `exp.py` 让后续同尺寸分配返回一个受控 largebin chunk 的 metadata 区，从而可改写它的 `fd_nextsize/bk_nextsize`。脚本还利用程序打印的分配地址判断页低位并重试，避免把错误 safe-linking 掩码带入后续操作。

### 3. largebin attack 改写 stdout 指针

把 largebin chunk A 的 nextsize 链指向 libc 中的 `stdout` 指针变量附近，再释放并插入更小的 chunk B。largebin 插入写把 `stdout` 变量改成 chunk B 的 header 地址。B 在释放前已经按 `_IO_FILE`、wide data 和伪 wide vtable 布局好，因此程序下一次输出 `blade: ...` 时会从伪 FILE 进入宽字符处理链。

伪结构最终调用 `setcontext`，恢复的 `rsp` 指向另一块堆内 ROP。ROP 使用配套 libc 的 gadget 完成：

```text
open("flag", 0, 0)
read(3, buffer, 0x100)
write(1, buffer, 0x100)
```

这样无需通过题目隐藏的 table-call CFI，也不依赖 shell。

### 4. 识别并排除干扰 gate

父进程确实会在第 35、45、58 次 table-call 检查固定 hash，相同 hash 还对应多个寄存器效果不同的 entry。单纯把 hash 映射回任意地址只能到达 gate，不能证明 syscall 参数受控。由于 stdio 链未被拦截，继续投入时间求整条 CFI 状态机反而扩大了不必要的约束；官方脚本的成功路径也直接验证 FSOP 才是决定性解法。

## 方法总结

本题需要区分“可触发的隐藏机制”和“完成目标的最短可靠原语”。自合并 UAF 同时提供 unsorted 泄漏与跨 bin 双重释放，tcache poisoning 再把写能力提升到 largebin metadata，最后通过 stdout 指针和 FSOP 接到 setcontext/ORW。每一步都能独立观察：merge 输出有 libc 指针、目标分配命中伪地址、stdout 指向伪 FILE、最终 ORW 输出 flag。CFI gate 没有形成等价的参数控制证据，因此应作为干扰项说明，而不能冒充主利用链。
