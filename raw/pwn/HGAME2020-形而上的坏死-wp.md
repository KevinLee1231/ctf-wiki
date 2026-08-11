# 形而上的坏死

## 题目简述

程序把多种弱点藏在分轮交互中：第一轮提供一次任意写和 6 字节泄露；数组下标可为负数；某个字符串被当作格式化串；最后长度检查又混淆了有符号与无符号整数。完整利用需要依次泄露 PIE、canary 和 libc，再构造向低地址生长的栈溢出 ROP。

## 解题过程

第一轮可部分覆盖返回地址的低 2 字节，使函数返回 `main`。因为 PIE 的高位在同一进程内不变，这种 partial overwrite 可以在不知道完整基址时重新进入程序；再借助 6 字节读泄露代码地址，按题目构建减去 `0x990` 得到 PIE 基址。

程序随后用用户控制的索引访问全局数组，未检查负数。根据数组基址与 GOT 表项的距离计算负索引，把 `puts@GOT` 改为 `printf@PLT`。下一次原本执行 `puts(user_string)` 的位置就变成：

```c
printf(user_string);
```

在“golden fish specimen”字段提交位置参数格式串，逐个确认栈布局后泄露 canary 和 libc 指针，例如：

```text
%<canary_index>$p.%<libc_index>$p
```

最后一轮把输入长度先按有符号数接收，却在后续按无符号值参与边界或地址计算。提交负数 `-2147000038` 可令写入位置向栈低地址越界，覆盖保存的返回现场。构造 ROP 时保留刚才泄露的 canary，并使用 PIE 中的 `pop rdi; ret`、libc 中的 `/bin/sh` 和 `system`：

```python
payload  = padding
payload += p64(canary)
payload += p64(saved_rbp)
payload += p64(pop_rdi)
payload += p64(bin_sh)
payload += p64(system)
```

题目二进制中可用 gadget 的相对偏移为 `0xf93`，`/bin/sh` 相关的题内偏移记录为 `0x2020e0`；最终地址仍应分别加上已泄露的模块基址。触发返回即可进入 shell 并读取 flag。

原题还存在概率型捷径：把返回地址低字节改到 `__libc_start_main` 附近，重新进入并结合 one-gadget；它依赖约 $1/16$ 的地址随机化命中率，稳定性低于上面的完整泄露链。

## 方法总结

- 核心链路：返回地址 partial overwrite 泄露 PIE、负索引改 GOT、格式化字符串泄露 canary/libc、有符号转换触发栈溢出。
- 识别信号：同一输入值跨越 signed/unsigned 类型，或索引只检查上界不检查下界时，应追踪最终指针运算。
- 复用要点：分阶段利用要让每轮泄露为下一轮提供基址；格式化参数序号、gadget 和偏移必须按实际构建验证。
