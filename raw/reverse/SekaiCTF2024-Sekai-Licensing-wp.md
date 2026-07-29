# Sekai Licensing

## 题目简述

题目模拟硬件绑定 DRM。附件包括 Windows 客户端和一次成功认证的 WebSocket 流量，要求恢复许可证与绑定硬件，并让本机客户端通过服务器下发的 Challenge VM。

完整链条分为三层：先破解因弱随机数而失效的 libhydrogen 密钥交换并解密 PCAP；再逆向许可证响应包，恢复正确许可证和硬件指纹；最后在用户态拦截 CPUID、`NtQuerySystemInformation` 和 `KUSER_SHARED_DATA` 访问，同时绕过程序自身的 `.text` 完整性哈希。

## 解题过程

### 1. 识别 WebSocket 加密与弱随机数

客户端字符串表暴露了 IXWebSocket。它为所有消息使用同一回调，在主函数中可以定位到体量最大的消息处理函数。解开其中的 SSE 字符串混淆后，关键标识为：

```text
Noise_Npsk0_hydro1
```

该字符串、`hydro_kx_n_1` 风格的 48 字节首包以及 secretbox 帧格式共同表明程序使用 libhydrogen 的 Noise N 变体。二进制还给出服务器静态公钥；secretbox 的 8 字节 context 为：

```text
SekaiCTF
```

密码原语本身没有问题，问题在客户端的 `hydro_random_context` 初始化。主函数只取 `rdtsc` 最低 8 位作为 seed，再通过固定移位/XOR 递推生成 48 字节状态，因此候选流只有 $2^8=256$ 个。

对每个 seed 重现这段错误 reseed，再调用 `hydro_kx_n_1` 与服务器公钥生成会话密钥。用候选接收密钥解密服务器的许可证请求；secretbox 自带 MAC，首次认证成功的候选就是正确 seed：

```text
seed = 0xe5
```

恢复出的客户端方向密钥为：

```text
Send key:
7BDF41B5BC2A253E512494B17F7BDA8A5A133500398063B5A4B880979987AF30

Receive key:
7193B719C4A1E76BE2BDA9EA42B8302CD845A0B1F645A71536B01C67932D991A
```

随后即可用 `hydro_secretbox_decrypt(..., msg_id=0, context="SekaiCTF", key)` 解密后续两个 WebSocket 帧。服务器发出的明文只有 `00`，即许可证请求 opcode 0；客户端的大响应以 `01` 开头，即许可证响应 opcode 1。

### 2. 解析许可证与硬件指纹包

许可证响应并非直接序列化字段。opcode 后先存一个取反的 64 位乘法逆元，随后不同字段分别经过位翻转、旋转、异或、加减和奇数乘法。官方 `sekai-packet-decryptor` 已给出三类逆变换：

```cpp
serial = reverse_bits(x) - 0xcec942ea3098af2c;
serial *= mod_inverse;
serial ^= rol(serial, 16) ^ ror(serial, 42);

misc = x ^ ror(x, 8) ^ rol(x, 14);
misc ^= 0x9dac012771735241;
misc *= mod_inverse;
misc = reverse_bits(misc);

cpuid = x + 0x87e3189b7d1f5464;
cpuid ^= ror(cpuid, 12) ^ rol(cpuid, 36);
cpuid = reverse_bits(cpuid);
cpuid *= mod_inverse;
```

按客户端写入顺序解析后，包结构为：

```cpp
struct LicensePacket {
    uint8_t  opcode;                 // 1
    uint64_t encoded_mod_inverse;
    uint64_t serial_key[4];          // 32 字节
    uint64_t number_of_processors;
    uint64_t firmware_type;
    uint8_t  boot_identifier[16];
    uint64_t nt_minor_version;
    uint64_t nt_major_version;
    uint64_t nt_build_number;
    uint64_t image_base_address;
    uint64_t cpuid_brand_parts[12];
};
```

其中硬件来源分别是：

- `NtQuerySystemInformation(SystemBasicInformation)` 的 `NumberOfProcessors`；
- `SystemBootEnvironmentInformation` 的 `FirmwareType` 与 `BootIdentifier`；
- 固定映射 `0x7ffe0000` 的 `KUSER_SHARED_DATA` 中，偏移 `0x260/0x26c/0x270` 对应 build、major、minor；
- PEB 的 `ImageBaseAddress`；
- CPUID 叶 `0x80000002` 至 `0x80000004` 返回的 48 字节 Processor Brand String。

最终恢复值如下：

| 字段 | 正确值 |
| --- | --- |
| License key | `21D4BA14519138F6E0FE409C6EE31EC59496E4DE18F17FD4949306FA7FE5CE79` |
| Number of processors | `192` |
| Firmware type | `1`（BIOS） |
| Boot identifier | `F35D3C35AFCFB146EB6DAB4FE2E00CC2` |
| NT major version | `9` |
| NT minor version | `42`（`0x2a`） |
| NT build number | `4919`（`0x1337`） |
| Image base | `0x7ff7b0510000`（服务器不校验） |
| CPU brand string | `Adv. Miku Devices Ultra 192-Core Processor` |

官方说明的某处输出把 major/minor 标签写反了；服务器 `CorrectConfiguration`、`KUSER_SHARED_DATA` 标准偏移以及官方 patcher 都一致确认正确值是 major 9、minor 42。

### 3. 为什么只伪造网络包还不够

服务器验证许可证与配置后下发 opcode 2，即 Challenge VM 字节码。VM 的操作数会使用上面这些硬件值参与解码，并通过专用 opcode 再次读取 CPUID、处理器数量、启动环境和 Windows 版本。若只把出站网络包改成正确配置，而客户端执行时仍读取本机真实硬件，VM 会得到错误结果，服务器最终返回 `Hardware challenge/response failed.`。

因此必须让客户端运行期间的每一次硬件访问都看到恢复出的配置。官方解法由三个组件组成：

1. C# 扫描器用 AsmResolver 读取 PE，并用 Iced 沿真实指令边界查找 CPUID、目标 syscall 和 `KUSER_SHARED_DATA` 引用，生成 `offsets.json`；
2. launcher 以正确许可证参数创建挂起进程，注入 patcher DLL，再恢复主线程；
3. patcher DLL 安装 VEH、克隆原始 `.text`，并应用 `offsets.json` 中的补丁。

挂起启动命令等价于：

```text
sekai-licensing.exe 21D4BA14519138F6E0FE409C6EE31EC59496E4DE18F17FD4949306FA7FE5CE79
```

### 4. 用 UD2 和 VEH 模拟 CPUID

CPUID 指令只有两个字节 `0f a2`，原地无法装下返回四个寄存器的 hook。扫描器定位每条 CPUID 后，patcher 把它替换为同样两字节的 `UD2`：

```text
0f a2  ->  0f 0b
```

VEH 捕获 `EXCEPTION_ILLEGAL_INSTRUCTION`，检查异常点的标记字节。若输入叶为 `0x80000002`、`0x80000003` 或 `0x80000004`，就把正确品牌字符串的 12 个 dword 分批写回 `RAX/RBX/RCX/RDX`；其他 CPUID 叶调用真实 `__cpuid`。最后令 `RIP += 2` 继续执行。

这种做法不需要内核驱动或自建 hypervisor，也不会受两字节原指令长度限制。

### 5. 重写 KUSER_SHARED_DATA 读取

`KUSER_SHARED_DATA` 在用户态只读，不能直接改写。Iced 的 `IPRelativeMemoryAddress` 能识别每条访问：

```text
0x7ffe0260 -> NtBuildNumber = 0x1337
0x7ffe026c -> NtMajorVersion = 9
0x7ffe0270 -> NtMinorVersion = 42
```

扫描器根据原助记符生成等价立即数指令。例如原先的 `mov edx,[0x7ffe0260]` 改成 `mov edx,0x1337`；`add ecx,[0x7ffe0270]` 改成 `add ecx,42`。它覆盖 `mov/movzx/add/sub/and/or/xor/imul` 等形式，若新指令更短则用 NOP 补齐，保证后续地址不移动。

### 6. 拦截 NtQuerySystemInformation

客户端使用直接 syscall，传统用户态 API hook 看不到调用。扫描器先找到保存 `NtQuerySystemInformation` syscall ID 的全局位置，再沿指令流找到对应 `syscall`，把两字节指令替换为非法标记 `0f ff`。

VEH 根据 `R10` 中的第一参数 `SYSTEM_INFORMATION_CLASS` 分流：

- `SystemBasicInformation`：先调用真实系统调用，再把 `NumberOfProcessors` 改成 192；
- `SystemBootEnvironmentInformation`：改成 BIOS，并复制正确的 16 字节 Boot GUID；
- `SystemKernelDebuggerInformation`：返回 `KernelDebuggerEnabled=false`、`KernelDebuggerNotPresent=true`，同时满足反调试检查。

处理完成后把 NTSTATUS 放进 `RAX`，令 `RIP += 2` 跳过非法指令。

### 7. 保留未修改的 .text 供完整性哈希

上述补丁都会改变 `.text`，而 Challenge VM 最后还会以 context `Miku:3:3` 对原程序代码做哈希。patcher 必须在写补丁前复制整个 `.text`。在 RVA `0x3d488`，原程序把待哈希区起点装入 `RCX`；把这段七字节指令替换成 `UD1` 加 NOP，VEH 遇到专用标记后将 `RCX` 改成克隆区地址，再跳过七字节。

这样实际执行的是已打补丁代码，而自校验读取的仍是原始字节，VM 的最终 32 字节 hash 与服务器预期一致。

完成全部 spoof 与自校验绕过后，客户端输出：

```text
Successfully authenticated user of license key.
Flag: SEKAI{m1ku_l0ves_cpu1d_and_ku3r_shar3d_data}
```

## 方法总结

本题把网络密码、硬件指纹和虚拟机绑定在同一条验证链上。`rdtsc` 只保留 8 位使安全的 libhydrogen 协议退化为 256 个候选；解密 PCAP 后可恢复许可证和完整目标硬件，但服务器下发的 VM 又迫使客户端在执行时持续呈现同一配置。

最终方案以“扫描真实指令边界—原长替换为非法指令—VEH 模拟语义”为核心：CPUID 和直接 syscall 走异常处理器，只读共享页访问改成同长度立即数运算，原始 `.text` 克隆用于通过哈希。官方材料中的 7 张图片均只是代码、日志或网页文字截图，相关信息已完整转写到正文，没有保留无视觉价值的图片依赖。
