# CakeCTF2021 Not So Tiger

## 题目简述

程序用 `std::variant<Bengal, Ocicat, Ocelot, Savannah>` 保存四种猫。`Ocelot::set` 使用无长度限制的 `strcpy` 写入 0x20 字节数组，能越过对象末尾覆盖 `variant` 的类型判别值。不同类型的首字段布局又不相同，因此可以把原本的整数解释成指针，构造任意地址读；菜单中新建猫使用的栈缓冲区还提供了最终栈溢出。

## 解题过程

### 覆盖 variant 判别值并构造任意读

`Ocelot` 的布局为：

```cpp
class Ocelot {
    long age;          // offset 0x00
    char name[0x20];   // offset 0x08
};
```

先创建类型 2 的 Ocelot，再调用 `set(target, "A" * 0x20)`。`strcpy` 写入 0x20 个 `A` 后还会补一个 NUL，这个 NUL 正好把紧随对象存储区的 `variant` 判别值改为 0，即 Bengal。

Bengal 的首字段是 `char *name`，所以之前放在 Ocelot `age` 中的 `target` 地址会被当作字符串指针。菜单的 Get 操作访问 Bengal 后，便从任意目标地址开始输出数据。

### 逐步泄露 libc、栈和 canary

利用任意读依次完成：

1. 读取主程序 `stdout` 指针，减去 `_IO_2_1_stdout_` 偏移得到 libc 基址。
2. 读取 libc 的 `environ`，取得栈地址。
3. 读取已知栈偏移处的 canary。canary 首字节为 NUL，所以从 `canary + 1` 读取 7 字节后再左移 8 位还原。

每次读取前重新创建 Ocelot，再用相同的判别值覆盖流程设置目标地址，避免前一次错误类型状态影响后续操作。

### 利用新建菜单的栈溢出

`New cat` 分支中还有 `char name[0x20]`，输入同样没有宽度限制。官方附件布局下，填充 0x88 字节到 canary，保留已泄露的 canary 后构造 ROP：

```text
padding | canary | saved values | ret | pop rdi | &"/bin/sh" | system
```

选择退出菜单触发函数返回，ROP 调用 `system("/bin/sh")`，最终读取：

```text
CakeCTF{c4n_U_d15t1ngu15h_b3tw33n_th353_c4t_5p3c13s?}
```

具体栈偏移和 gadget 地址依赖发布二进制，复现时应从附件重新确认。

## 方法总结

- 带标签联合体的安全性不仅依赖对象内容，还依赖类型判别值不被越界写破坏。
- 利用不同类的字段布局差异，可以把整数和指针相互混淆，形成可重复的任意读。
- 本题是“类型混淆泄露信息，再用普通栈溢出拿执行”的组合链，两个漏洞各自承担不同阶段。
