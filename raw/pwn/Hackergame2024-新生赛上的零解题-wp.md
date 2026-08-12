# 新生赛上的零解题

## 题目简述

题目用两个 AMD64 程序展示用户态返回地址保护及其绕过。第一问是无 PIE 的静态程序：`vuln` 把当前返回地址复制到一块固定映射的“软件影子栈”，却向仅有两个 `size_t` 的局部数组连续写入最多 32 个整数；返回前若地址不一致就 `throw 1`。第二问由 Intel SDE 以 CET 模式模拟 Shadow Stack/IBT；程序仍有数字数组溢出，并提供读取非黑名单文件和任意地址改一字节两个 gift。

两问的决定性目标都是从栈溢出、C++ 异常处理和影子栈语义构造控制流/内存破坏 primitive，明确归入 `pwn`。

## 解题过程

### Canary Bypass：借异常路径跨过返回检查

第一问 `vuln` 的核心逻辑是：

```cpp
stack_buf[ssp++] = *(size_t *)(rbp + 8);
size_t buf[2];

for (int i = 0; i < 32; i++) {
    scanf("%lu", &temp_num);
    if (temp_num == 0x31337) break;
    buf[i] = temp_num;
}

if (*(size_t *)(rbp + 8) != stack_buf[ssp - 1])
    throw 1;
```

普通改返回地址会触发 `throw`，但 C++ 异常展开会依据元数据进入匹配的 `catch`，不等价于执行被覆盖的 `ret`。将 `vuln` 栈上的返回地址改成 `banner` 函数 try 区域内部地址 `0x401f0f`，异常展开就会把它归属到 `banner` 的 landing pad，从而改走对应 catch。这类通过异常处理元数据拼接控制流的技术常称 CHOP（Catch Handler Oriented Programming）。

题目 `setup` 把一块可读写区域固定映射到 `0x31336000` 附近，并把伪影子栈指针设在其中。利用分三轮：

1. 第一轮仅推动伪影子栈状态，保持循环继续。
2. 第二轮覆盖保存的 `rbp` 与返回地址，使异常路径转入 `banner` 的 catch，并把后续栈迁移到固定映射区 `0x31337000`。
3. 第三轮把 `/bin/sh\0` 和完整 syscall ROP 链写到可预测的新栈，设置 `rax=59`、`rdi` 指向字符串、`rsi=rdx=0` 后执行 `syscall`。

静态链接且无 PIE 让 gadget、landing pad 和固定映射地址保持稳定。虽然 `checksec` 显示 canary，题目使用的旧版 GCC 在这一异常抛出路径上没有先执行有效的 canary 检查，因此异常路径仍可利用。

### CET Bypass：十进制解析失败保留栈字节

第二问使用新版 GCC，`throw` 前会检查 canary，直接写连续整数会破坏它。关键差异是代码直接执行 `scanf("%lu", &buf[i])`，却不检查返回值。输入单个 `+` 时，转换失败，`scanf` 不向目标地址写数据，但循环下标仍然增加。因此可以让 payload 中需要保留的 canary、保存寄存器等位置发送 `+`，只在指定槽位写入目标整数。

payload 的目标状态为：

- 覆盖正常栈上的返回地址/异常归属地址为 `banner` 的 try 内部地址 `0x401913`；
- 把 `main` 的 `win_buf` 改成 `/bin/sh\0`，因为 `banner` 的 catch 会执行 `system(data)`；
- 写入 `0x31337` 触发两个 gift 和 `throw win_buf`。

在没有 CET 时这已经足以劫持流程；SDE 的影子栈仍保存原返回地址，返回时会阻止篡改。

### 定位并同步 SDE 影子栈

gift1 禁止的仅是 `/flag`、`/flag1`、`/flag2` 解析后的精确路径，却允许读取 `/proc/self/maps`。从 maps 中找到 SDE 为 shadow stack 建立的映射基址，再按参考环境布局定位保存目标返回地址的槽位；官方 exploit 在给定 SDE 布局中使用该映射基址加 `0x2fd8`。

gift2 提供任意地址单字节写。把影子栈中对应返回地址的低字节改成目标 `0x401913` 的低字节 `0x13`，使普通栈与影子栈对同一次返回达成一致。由于两地址位于同一代码区域，高位相同，一字节写足够。之后异常进入 `banner` 的 catch，参数指向已布置的 `/bin/sh`，最终调用 `system`。

必须强调：这里能直接普通写 SDE 的 shadow-stack 映射，是教学模拟器实现提供的条件。Linux 内核真实 CET 用户态影子栈通常不能被普通 store 修改，需要 WRSS 等专用机制；不能把本题原样当作真实系统通用绕过。

### 验证

官方两份 exploit 分别固定了 CHOP landing pad/ROP 地址，以及 `+` 占位、`/proc/self/maps` 泄漏和单字节影子栈修补流程。本次未连接远程服务，也未启动 Intel SDE；验证限于源码、官方脚本与题解描述的静态一致性核对。

## 方法总结

- 核心技巧：第一问利用 C++ 异常展开改变控制流并迁移栈；第二问利用未检查的 `scanf` 失败跳过敏感槽位，再用 maps 泄漏和一字节写同步普通栈与 SDE 影子栈。
- 识别信号：栈溢出旁出现 `try/catch/throw`、异常 landing pad、SHSTK/IBT，以及输入转换返回值未检查时，不能只套普通 ROP 模板。
- 复用要点：先区分编译器软件检查、真实硬件 CET 与模拟器语义；解析失败是否推进循环但不写内存，是保留 canary 的关键；任何影子栈绕过都必须同时解释普通栈和保护副本为何一致。
