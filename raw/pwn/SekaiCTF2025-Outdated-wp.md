# Outdated

## 题目简述

题目是开启 PIE/ASLR 的 MIPS 程序。玩家输入的 `level` 是有符号整数，却被直接用作：

```c
unsigned short level_rewards[10];
level_rewards[level] = reward;
```

因此可在栈上相对该数组进行一次 2 字节越界写。`main()` 最后调用 `exit()` 而不是返回，所以覆盖保存的 `$pc` 或 `$s8` 不会生效。真正可利用的目标是栈上保存的 `$gp`。

## 解题过程

### 1. 为什么覆盖 `$gp`

MIPS PIC 代码用全局指针 `$gp` 作为 GOT 和全局数据寻址基准。函数调用后，编译器会从栈上保存位置恢复 `$gp`。官方解法使用负索引：

```text
level = -12
```

把一次 16 位写落到保存的 `$gp` 低位。随后程序调用 `printf()` 并恢复 `$gp`，后续 `puts()`、`exit()` 和 `.rodata` 参数地址都将以攻击者控制的新基址计算。

这比覆盖返回地址更有效，因为目标函数从不执行 `jr $ra`。

### 2. 在全局缓冲区伪造 GOT

全局 `game_name` 大小为 0x60，地址可由程序泄漏的 PIE 基址确定。把 `$gp` 部分改写到 `game_name` 附近后，正常 GOT 相对访问会落到攻击者填写的“伪 GOT”中。

伪 GOT 同时控制：

- 原 `puts` 调用的函数地址；
- 原 `exit` 调用的函数地址；
- 原本指向固定感谢字符串的参数地址。

### 3. 第一轮泄漏 libc 并返回 `main`

第一轮伪 GOT 设置为：

```text
puts 位置  -> puts_blue
puts 参数  -> real GOT[puts]
exit 位置  -> main
```

程序原本的：

```c
puts("Thanks for playing! Come again!");
exit(0);
```

因此变成“带颜色输出 `puts` 的真实 GOT 地址”，随后再次进入 `main()`。拿到 `puts` 地址后，可按附带 libc 符号偏移计算 libc 基址。

### 4. 第二轮调用 `system`

第二次进入程序后重新布置 `game_name`：

```text
puts 位置  -> system
puts 参数  -> "/bin/sh"
```

再次用 `level=-12` 部分覆盖保存的 `$gp`。最终同一处 `puts()` 调用被解释成：

```c
system("/bin/sh");
```

从而获得 shell。

仓库正式挑战镜像中的 flag 为：

```text
SEKAI{I've_dubb3d_th1s_t3chn1que_"GP_Overwrite"}
```

### 5. QEMU ASLR

官方环境对 QEMU ASLR 有自定义补丁，仍存在少量地址随机化。题目先给出程序地址泄漏以确定 PIE，而 libc 必须通过第一阶段再泄漏；这也是利用需要两次返回 `main`，不能直接一次调用 `system` 的原因。

## 方法总结

架构相关利用不能机械套用“覆盖返回地址”。MIPS PIC 中 `$gp` 决定 GOT 与 `.rodata` 寻址，且会在函数调用后从栈恢复；一次很小的 2 字节写也能把整个全局解析基准移到攻击者可控区。

分析此类题时应先确认函数终止方式，再沿后续真实会使用的寄存器和恢复序列寻找目标。若程序调用 `exit`，保存的 `$ra` 即使可写也可能毫无价值。
