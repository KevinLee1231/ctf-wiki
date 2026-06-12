# d3guard

## 题目简述

非常遗憾这题最终没有解，也许是出题上还有可以改进的空间，欢迎对UEFI PWN方面感兴趣的师傅私信交流！

题目是 UEFI PWN：QEMU 使用 `OVMF.fd` 作为 pflash 固件启动，`./content` 被挂载为 FAT 驱动器。普通启动会进入低权限 Linux 用户，无法读取 `/flag`；攻击目标是在 UEFI 阶段利用被 patch 的 `UiApp/d3guard` 漏洞，重新进入正常 UI 交互并修改启动参数，使系统以 root shell 启动。

## 解题过程

### 1. Analysis

观察启动脚本的参数可以发现，QEMU在启动时向pflash（可以看成是bios）写入了一个叫做OVMF.fd的固件，并且将./content 目录挂载为了一个fat格式的驱动器。熟悉UEFI开发的选手应该很快可以想到这是一个UEFI PWN，即通过UEFI环境下的漏洞利用完成提权

题目源文件的所有改动基于edk2项目：https://github.com/tianocore/edk2

EDK II 是 TianoCore 维护的 UEFI/PI 固件开发环境，提供构建 UEFI firmware、drivers、applications 的源码和基础库；OVMF（Open Virtual Machine Firmware）就是基于 EDK II 的 QEMU/虚拟机 UEFI 固件实现。因此本题逆向和利用时，需要按 EDK II 的 DXE/UiApp/Pool 分配模型理解固件行为。

运行启动脚本且不做任何操作将会直接进入操作系统，并切换到低权限用户。该用户没有根目录下flag文件的读权限。结合题目描述中的cat /flag 可以得知需要进行某种方式的提权以读取flag内容

```
/ $ ls -al /flag
-r--------    1 0        0               25 Feb 17 17:33 /flag
/ $ id
uid=1000 gid=1000 groups=1000
```

正常情况下，edk2会提供UI和EFI SHELL两种交互方式让用户运行EFI程序或者进行Boot参数的相关设置。检查boot.nsh 可以发现默认情况下内核的启动参数为：bzImage console=ttyS0

initrd=rootfs.img rdinit=/init quiet ，也就是说，如果我们能够进入UI或者EFI SHELL交互界面，然后修改Boot参数为bzImage console=ttyS0 initrd=rootfs.img rdinit=/bin/ash quiet就可以以root shell的方式进入操作系统，读取flag文件。

但是留意启动过程的输出会发现，进入EFI SHELL前的倒计时直接被掠过了（因为我把入口逻辑patch掉了）。于是只能尝试去进入UI交互界面。edk2进入UI交互界面的快捷键为F2（或F12），在启动时长按该按键即可进入UI交互程序。然而在本题中，并不会直接进入Ui交互界面，而是先进入了d3guard子程序，如下：

```
BdsDxe: loading Boot0000 "UiApp" from Fv(7CB8BDC9-F8EB-4F34-AAEA-
3EE4AF6516A1)/FvFile(462CAA21-7614-4503-836E-8AB6F4662331)
BdsDxe: starting Boot0000 "UiApp" from Fv(7CB8BDC9-F8EB-4F34-AAEA-
3EE4AF6516A1)/FvFile(462CAA21-7614-4503-836E-8AB6F4662331)
```

程序启动后有 `D^3 Guard` 字符画菜单，`1. Administrator 2. Visitor` 处输入超长内容时，用户名回显会泄露栈上地址；后续利用把这个栈地址作为覆盖返回地址的定位依据。

### 2. Reverse

现在首要任务就是对UiApp 进行逆向分析寻找能够进入正常Ui交互的方式。借助一些工具可以轻松地将UiApp 模块镜像提取出来，这里使用的是：https://github.com/yeggor/uefi_retool

`uefi_retool` 是一个 UEFI firmware reverse engineering 工具，仓库 README 给出的核心功能包括：从固件中提取 executable images 到 `modules` 目录、对整个固件做信息分析并输出 JSON、提取 proprietary protocol 列表，并提供 IDA 插件辅助分析。这里用它的 `get-images` 能直接从 `OVMF.fd` 中拆出 `UiApp` 模块，减少手工解析 firmware filesystem 的工作量。

通过逆向可以发现两个主要的漏洞，一个是尝试用Administrator身份登录时，存在一个格式化字符串漏洞，该漏洞可以泄露栈上的地址信息，包括镜像地址和栈地址：

一些队伍由于没注意到关于这个漏洞的hint导致差一点没拿到flag，深感可惜😭！！！

输入一串 `%p` 能看到格式化字符串没有被当作格式串执行，提示该方向不是本题主线；真正可用的是 Visitor 路径中的堆对象覆盖。

还有一个漏洞是在编辑用户描述信息的时候存在堆溢出（这一点大部分队伍都发现了）：

反编译结果显示 `visitor->desc` 分配大小为 `0x6164`，但后续编辑时可写入 `0xE164` 字节，能够从 `desc` 覆盖到相邻堆块元数据。

除了对于UiApp 镜像的逆向分析，还需要阅读edk2中AllocatePool的具体实现方式，这关系到漏洞利用的一些细节，这部分暂时省略

相关代码位于：https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Mem/Pool.c

`Pool.c` 是 EDK II DXE Core 的内存池实现，包含 `AllocatePool`/`FreePool` 相关逻辑以及 `POOL_HEAD`、`POOL_TAIL` 这类元数据处理。本题堆溢出能否把 chunk 放入指定 free list、能否借双向链表完成写原语，都依赖这里的 pool header、memory type 和释放链表行为。

### 3. Exploit

通过动态调试发现，1. New Visitor 之后，visitor->name 和visitor->desc 位于相邻的内存区间上，将两者调换位置让visitor->desc 位于低地址处，即可通过堆溢出漏洞覆盖visitor->desc 的POOL_TAIL 和visitor->name 的POOL_HEAD

主要关注POOL_HEAD结构体

```
typedef struct {
  UINT32             Signature;
  UINT32             Reserved;
  EFI_MEMORY_TYPE    Type;
  UINTN              Size;
  CHAR8              Data[1];
} POOL_HEAD;
```

结合对AllocatePool相关源代码的阅读，发现当调用FreePool 函数时，edk2会根据POOL_HEAD-

>EFI_MEMORY_TYPE 的不同而将堆块放入不同的链表中，而分配visitor->name 和visitor->desc 时， AllocatePool参数所用的EFI_MEMORY_TYPE 为EfiReservedMemoryType （即常数0）。如果通过溢出修改visitor->name 的POOL_HEAD->EFI_MEMORY_TYPE 为别的值，即可将其放入其它链表中，再次申请也不会被取出。

`AllocatePool` 的关键点是按 `EFI_MEMORY_TYPE` 选择不同链表。把 `visitor->name` 对应 `POOL_HEAD->EFI_MEMORY_TYPE` 从 `0` 改成目标类型后，释放时会进入另一条 free list，从而避开原本的再分配路径。

最后在4. Confirm && Enter OS 中还会分配一次堆内存，用于拷贝visitor->name 和visitor- >desc 并保存。这时候AllocatePool() 所申请的EFI_MEMORY_TYPE 为EfiACPIMemoryNVS （即常数10）。

因此最终把 `visitor->name` 的 `EFI_MEMORY_TYPE` 伪造成 `10`，再释放并劫持 free list 的 `FD/BK` 指针，就能在下一次 `AllocatePool(0xE164, 10, ...)` 时完成任意地址写。

结合上面的分析，将visitor->name 的POOL_HEAD->EFI_MEMORY_TYPE 设置为10，并将其Free。此时原先分配给visitor->name 的堆块进入了空闲链表（这是个双链表），通过劫持双链表的FD和BK指针可以向任意地址写一个自定义的值。结合最开始泄露出的栈地址，我们可以将d3guard函数的返回地址覆盖掉以劫持程序流。

实际上最后一步的解法是开放性的，只要达到劫持控制流的目的就行

由于d3guard() 的上层函数_ModuleEntryPoint+718 的位置会判断d3guard() 的返回值以决定是否进入UI交互界面，所以最直接的做法是覆盖d3guard返回地址跳过if分支直接进入UI交互界面。但是实际编写脚本时发现泄露出的程序地址与跳转的目标地址偏移不是很稳定（但是概率很大），于是覆盖

d3guard返回地址为一个栈上shellcode的地址（栈上没开NX防护），shellcode可以在输入Admin pass key时提前部署。借助shellcode以及寄存器中的镜像地址，可以计算出稳定的跳转目标地址。

成功进入Ui交互界面后，只需要通过操作菜单添加一个新的启动项，并将参数rdinit 设置为/bin/sh然后通过其进入操作系统，即可获得root权限。

开始没想到加启动项这个步骤也能成为一个坑点...其实可以编译一份原版OVMF.fd，进入Boot Maintenance Manager ，进入Boot Options ，选择Add Boot Option ，选择内核镜像

bzImage ，设置启动项名称rootshell ，设置内核启动的附加参数console=ttyS0

initrd=rootfs.img rdinit=/bin/sh quiet ，最后返回主页面选择启动项菜单，找到

rootshell 这一项

题目附件和利用脚本：https://github.com/yikesoftware/d3ctf-2022-pwn-d3guard

该仓库说明其内容是 D^3CTF 2022 `pwn-d3guard` 的附件和官方中英文 writeup，并标注了 `edk2`、`uefi-pwn`、`d3guard` 等主题；README 还提醒附件环境与比赛环境不完全一致。复现时应以仓库里的附件、利用脚本和中英文 writeup 对照本文流程确认偏移与环境差异。

## 方法总结

- 核心技巧：先用格式化字符串泄露镜像基址和栈地址，再利用 visitor 描述信息的堆溢出修改 `POOL_HEAD->EFI_MEMORY_TYPE`，把目标 chunk 放入后续会再次使用的 EDK II pool free list。
- 利用链：篡改 free list 的 FD/BK 获得任意写，覆盖 `d3guard` 返回地址；若直接跳 UI 分支偏移不稳定，可把返回地址指到栈上 shellcode，再由 shellcode 计算稳定目标地址。
- 复用要点：UEFI PWN 题要同时看固件启动链、UI/EFI Shell 入口是否被 patch、EDK II pool allocator 元数据，以及最终能否通过 boot option/kernel cmdline 达到 OS 层提权。
