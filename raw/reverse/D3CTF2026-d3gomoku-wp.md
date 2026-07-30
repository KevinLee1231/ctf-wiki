# d3gomoku

## 题目简述

题目给出一个 64 位 Windows 内核驱动 `drv_game.sys`，题面只有一句：

> Humans cannot defeat AI unless...

附件并不是普通的五子棋程序。第一阶段驱动负责设备协议、棋盘和历史记录；它还藏有一个加密 PE，运行时会把第二阶段作为内存模块启动。第二阶段是一个 Intel VMX hypervisor，它用 EPTP switching 把第一阶段的一段校验代码替换成隐藏版本。真正的目标不是按正常策略赢过 AI，而是：

1. 完成驱动握手；
2. 通过加载一个新的 `notepad.exe` 触发第二阶段；
3. 在 Intel VMX/EPT 环境中让十轮人类/AI 落子历史满足两组 64 位证明；
4. 避开最后一个只取 3 bit 的滚动哈希门；
5. 用十轮具体坐标解密第二阶段内的 39 字节 flag；
6. 让驱动把游戏结果写成“人类胜利”，并从注册表读取 `RemoteFlag`。

官方提示要求在 Intel 平台运行，并推荐 Windows 10 Version 19045；这不是兼容性建议，而是题目逻辑的一部分。第二阶段直接执行 `VMXON`、`VMPTRLD`、`VMLAUNCH`、`VMREAD`、`VMWRITE` 和 `VMFUNC`，AMD 主机不能原样执行这条链。比赛平台入口为 [D3CTF Race](https://race.d3c.tf/)。

文件指纹如下：

| 文件 | SHA-256 |
| --- | --- |
| `drv_game.sys` | `3a68b0d9ae7cab8506d81360d68446a27c8ce443280f8ea16237d58a79288d17` |
| 解出的 `stage2.dll` | `daeb34debb1df5c917b0e3455a00ff9e429d927d847e4a9bc5030a2f018e6fb6` |

`drv_game.sys` 带有 `CN=D3Gomoku Test Cert` 的测试签名，但证书链默认不受信任。动态验证应放在快照可恢复的 Intel Windows 10 虚拟机内；不要在日常主机上开启测试签名、关闭 VBS/Hyper-V 或加载未知内核驱动。

本次给定机器是 AMD Ryzen + Windows 11，无法执行题目要求的 Intel VMX 路径。因此本文完成的是：

- IDA Pro MCP 静态还原；
- 第二阶段提取；
- Unicorn 对第一阶段关键路径的指令级验证；
- 隐藏证明的数学反解；
- 直接调用第二阶段原生例程完成 39-bit MITM 和内置 SHA-256 校验；
- 可在 Intel Windows 10 虚拟机中运行的客户端和 WinDbg 脚本。

最终恢复并通过第二阶段内置摘要校验的 flag 为：

```text
d3ctf{Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100}
```

## 解题过程

### 1. 第一阶段：设备与 IOCTL 协议

驱动的首要接口是：

```text
\Device\D3Gomoku
\DosDevices\D3Gomoku
用户态路径：\\.\D3Gomoku
```

文件对象上下文中有一个会话状态：

- `0`：刚打开设备；
- `1`：握手成功；
- `2`：`notepad.exe` 触发完成，游戏和第二阶段可用。

`DispatchDeviceControl` 位于 `drv_game+0x319C0`。第一次握手使用 IOCTL `0x222004`，输入恰好是 32 字节：

```c
struct Handshake {
    uint32_t magic;       // 0x31414744, little-endian "DGA1"
    uint32_t version;     // 1
    uint32_t size;        // 0x20
    uint32_t nonce;       // 本文固定为 0x12345678
    uint32_t checksum;    // 0xA40A63E3
    uint32_t reserved[3]; // 全 0
};
```

校验和由 `drv_game+0x1240` 计算。对上述固定字段，结果是：

```text
nonce    = 0x12345678
checksum = 0xA40A63E3
```

握手只会把状态从 `0` 改为 `1`。之后必须新启动一次 `notepad.exe`。镜像加载回调识别到完整文件名后，把对应文件上下文改成状态 `2`，并启动内存中的第二阶段模块。

游戏 IOCTL 如下：

| IOCTL | 输入 | 输出 | 作用 |
| --- | --- | --- | --- |
| `0x222040` | `<0x474F4D4B, 0>` | 8 字节 | 重置并返回 `game_id` |
| `0x222044` | `<0x52534554, game_id, 0>` | `0xEC` 字节 | 开始本局 |
| `0x222048` | `<0x4D4F5645, game_id, x, y, 0>` | `0xEC` 字节 | 人类落子并让 AI 落子 |
| `0x22204C` | undo 请求 | — | 悔棋 |
| `0x222050` | resign 请求 | — | 认输 |
| `0x222054` | 无 | `0xE0` 字节 | 状态/遥测，开头为 `MVXG` |

落子回复的前 8 字节是：

```c
struct MoveReply {
    uint32_t result; // 0=继续，1=拒绝，2=人类胜，3=AI 胜
    uint8_t ai_x;
    uint8_t ai_y;
    uint16_t reserved;
};
```

后面跟 15×15 棋盘。`solve_client.py` 完整实现了这套协议。

### 2. 解出内嵌的第二阶段

第一阶段在 `drv_game+0x325A0` 附近解密一段长度为 `0x1EF88` 的数据。密文位于 RVA `0xCCA50`，32 位 key 为 `0xA7C35E19`。逐字节算法是：

```python
plain[i] = (
    encrypted[i]
    ^ ((0xA7C35E19 >> (8 * (i & 3))) & 0xFF)
    ^ (((i + 1) * 0x11) & 0xFF)
)
```

对应脚本是 `extract_stage2.py`。在 WSL Kali 中复现：

```bash
source /home/kali/miniforge3/etc/profile.d/conda.sh
conda activate ctf-tools
cd "/mnt/d/文档/新建文件夹/D3CTF2026/d3gomoku"
python extract_stage2.py drv_game.sys -o stage2.dll
sha256sum stage2.dll
conda deactivate
```

解密结果以 `MZ` 开头，PE 签名和节表均正常。把它单独载入 IDA 后，可以看到两个导出：

```text
D3VmxEmbeddedEntry
D3VmxEmbeddedStop
```

本目录同时保存了两个已标注数据库：

- `drv_game-annotated.i64`
- `stage2-annotated.i64`

### 3. Intel VMX 与 EPT 隐藏代码

第二阶段的重要函数及 RVA 为：

| RVA | 标注名 | 作用 |
| --- | --- | --- |
| `0x10837` | `SwitchEptpVmfunc` | `VMFUNC(0, index)` 切换 EPTP |
| `0x10841` | `EptHookProofTrampoline` | 切回 EPTP 0 后进入公共证明校验 |
| `0x10A34` | `ComputeMoveHistoryProof` | 计算两组落子历史证明 |
| `0x11CDC` | `WrapInnerFlag` | 把 39 字节明文包装成 `d3ctf{...}` |
| `0x12034` | `CopyDecryptedInnerFlag` | 令牌和双摘要通过后复制明文 |
| `0x12644` | `IssueFlagReadToken` | 校验 flag 状态并生成读取令牌 |
| `0x12940` | `XorFlagBufferForRound` | 按本轮具体坐标异或 flag 缓冲区 |
| `0x12FFC` | `ValidatePlaintextSha256` | 比较内置的 32 字节明文摘要 |
| `0x131CC` | `ComputeCoordinateMix` | 混合人类/AI 的四个坐标 |
| `0x14C04` | `ComputeCodeAnchor` | 生成固定代码锚 `0xB582A7EB` |
| `0x1162C` | `BuildEptHookStub` | 生成入口 hook |
| `0x11AD8` | `BuildProofConstantTrampoline` | 生成带真实常量的影子页代码 |
| `0x15D38` | `BuildShadowVerifierPage` | 构造 EPTP 1 的影子校验页 |
| `0x16018` | `LocateStage1HookTargets` | 扫描 marker 和 stage1 包装器签名 |
| `0x16504` | `WriteRemoteFlagRegistryValues` | 写出 flag 状态和完整 flag |
| `0x168B0` | `ReadHypervisorProofSeed` | 通过被截获的 MSR 读取得到固定种子 |
| `0x171B0` | `InitializeVmxAllProcessors` | 在所有逻辑处理器上初始化 VMX |
| `0x17444` | `EnterVmxOnCurrentProcessor` | 当前处理器的 VMXON/VMCS/EPT 初始化 |
| `0x184F0` | `ValidateAndLatchProof` | 验证并把结果写回第一阶段上下文 |
| `0x18E90` | `ValidateAlternateProofConstants` | 原始页上的替代/诱饵常量 |
| `0x19140` | `VmExitHandler` | VM-exit 和 MSR 模拟 |

`LocateStage1HookTargets` 不是按固定模块基址硬编码地址，而是枚举 `drv_game.sys` 的可执行节并搜索两种签名：

1. `stage2+0x1DD90` 的十个混淆字节经 `stage2+0x11A0` 解码为 ASCII `ACELETMEIN`，唯一命中 `drv_game+0x3130E`。这十个字节被一次无条件跳转跨过，正好是 `CheapRollingHashGate` 中的内联 marker；
2. `stage2+0x1D778` 保存 16 字节固定模式，唯一命中 `drv_game+0x311C5`。第二阶段继续检查前五字节，最终恢复包装器入口 `drv_game+0x311C0`。

包装器内容为：

```asm
test    rcx, rcx
jz      fail
cmp     dl, 0Fh
ja      fail
cmp     r8b, 0Fh
ja      fail
sub     rsp, 28h
call    drv_game+0x430BE
add     rsp, 28h
movzx   eax, al
ret
fail:
xor     eax, eax
ret
```

第一阶段的隐藏胜利检查会调用这个包装器。正常 EPT 下，它只是进入 `CheckBoardGate`。第二阶段则把执行流改成：

```text
CheckHiddenHumanWin
  -> drv_game+0x311C0
  -> EPT hook stub
  -> VMFUNC 切到 EPTP 1
  -> 同一虚拟地址映射到影子 verifier 页
  -> 加载真实 proof target 到 R10/R11
  -> VMFUNC 切回 EPTP 0
  -> ValidateAndLatchProof
```

原始 verifier 页里的常量为：

```text
0x3CA5218358266D71
0x409B7C7DB5881627
```

直接静态分析第一阶段或只看 EPTP 0，会得到这组错误目标。`BuildProofConstantTrampoline` 生成的影子代码中才有真实值：

```text
target_1 = 0x44EA257DE1CEFB27
target_2 = 0xEED3C641A4C3A7A7
```

这也是官方要求 Intel 平台的真正原因：`VMFUNC` 切换后，同一个线性地址会看到另一页代码。

### 4. 反解十轮落子证明

`ComputeMoveHistoryProof` 首先检查：

- 文件上下文状态必须为 `2`；
- 游戏已初始化且结果仍为 `-1`；
- 人类与 AI 历史数量合法且相等；
- 请求携带的滚动哈希等于当前状态和哈希历史；
- VMX 侧保存的局号、历史进度和第一阶段一致。

VM-exit handler 对证明函数使用的两个 MSR 读进行模拟。最终：

```text
seed = 0xD10155 ^ 0xD31145 = 0x00021010
```

两个初始累加器使用 MurmurHash3 的 `fmix64`：

```python
initial_1 = fmix64(
    (((seed ^ 0xA5A5D3E7) | (seed << 32))
     ^ 0x6D7B9A49C2E5A13F)
)

initial_2 = fmix64(
    ((((~seed & 0xFFFFFFFF) << 32)
      | (((seed << 7) & 0xFFFFFFFF) ^ 0x9E3779B9))
     ^ 0xBB67AE8584CAA73B)
)
```

数值为：

```text
initial_1 = 0x99998E967E16F091
initial_2 = 0x749434B6121A32BC
```

设第 `i` 轮：

```text
人类坐标 H_i = (hx_i, hy_i)
AI 坐标   A_i = (ax_i, ay_i)
X_i = hx_i + ax_i
Y_i = hy_i + ay_i
```

初始前一轮坐标和来自 seed：

```text
X_-1 = (seed >> 1) & 0xF = 8
Y_-1 = (seed >> 5) & 0xF = 0
```

每轮常量和 base-37 digit 为：

```text
c1_i = (byte(seed, i & 3)       + 7*i  + 19) mod 37
c2_i = (byte(seed, (i+1) & 3)   + 11*i + 29) mod 37

d1_i = (X_(i-1) + X_i + c1_i + 2*Y_i) mod 37
d2_i = (Y_(i-1) + Y_i + c2_i + 3*X_i) mod 37

P_i = (37*P_(i-1) + d1_i) mod 2^64
Q_i = (37*Q_(i-1) + d2_i) mod 2^64
```

这里有一个很省事的逆向点。固定轮数 `n` 时：

```text
target = initial * 37^n + base37(digits)  (mod 2^64)
```

当 `n <= 12`，有 `37^n < 2^64`，所以：

```text
D = (target - initial * 37^n) mod 2^64
```

只有 `D < 37^n` 时，这个轮数才可能成立；随后直接把 `D` 展开成恰好 `n` 位 base-37。依次检查 `n=1..12`，第一次同时满足两组目标且坐标和方程有解的是 `n=10`：

```text
d1 = [13, 29, 28, 35, 30, 20, 4, 21, 8, 6]
d2 = [24, 23,  6,  6,  8,  5, 26, 6, 15, 8]
```

枚举每轮合法的 `X_i,Y_i ∈ [0,28]`，得到唯一坐标和序列：

```text
(5,1), (3,8), (3,12), (5,12), (0,18),
(0,12), (9,3), (8,5), (6,7), (5,4)
```

`proof_solver.py` 从目标常量重新推导上述结果，而不是把它们当作输出硬编码。脚本最后还会验证选定的具体落子不冲突、不越界、不产生普通五连，并精确命中两组 64 位目标。

坐标和还不足以得到 flag，因为第二阶段的逐轮解密使用四个具体坐标。后文的 39-bit MITM 会选出下面这组唯一合法分解：

| 轮次 | 要求的 `(X,Y)` | 人类落子 | 强制 AI 落子 |
| ---: | ---: | ---: | ---: |
| 1 | `(5,1)` | `(2,0)` | `(3,1)` |
| 2 | `(3,8)` | `(0,6)` | `(3,2)` |
| 3 | `(3,12)` | `(0,9)` | `(3,3)` |
| 4 | `(5,12)` | `(2,8)` | `(3,4)` |
| 5 | `(0,18)` | `(0,8)` | `(0,10)` |
| 6 | `(0,12)` | `(0,7)` | `(0,5)` |
| 7 | `(9,3)` | `(4,1)` | `(5,2)` |
| 8 | `(8,5)` | `(4,3)` | `(4,2)` |
| 9 | `(6,7)` | `(4,5)` | `(2,2)` |
| 10 | `(5,4)` | `(1,0)` | `(4,4)` |

这 20 个落点均不冲突，双方在十轮内都没有形成普通五连。人类无需让正常棋盘判定返回胜利；EPTP 1 上的影子 verifier 会让隐藏检查改用证明和 flag 状态。

复现静态证明：

```powershell
wsl bash -lc 'source /home/kali/miniforge3/etc/profile.d/conda.sh && conda activate ctf-tools && cd "/mnt/d/文档/新建文件夹/D3CTF2026/d3gomoku" && python proof_solver.py; task_status=$?; conda deactivate; exit $task_status'
```

关键输出：

```text
VMX seed:             0x00021010
initial proof #1:     0x99998e967e16f091
initial proof #2:     0x749434b6121a32bc
real target #1:       0x44ea257de1cefb27
real target #2:       0xeed3c641a4c3a7a7
smallest round count: 10
verified: legal board, no ordinary five, exact hidden proof targets
```

### 5. 从具体坐标恢复 39 字节 flag

证明函数还有一条容易漏掉的数据流。`ResetProofState` 在状态对象的 `+0x10` 放入 39 字节密文：

```text
3b0fe8d33280a674c76a422d65ba6724d5fb2abfb8f5f7fd1a9ac555d1fea28e89de2ea7ccc7e4
```

处理第 `i` 轮历史时，`ComputeMoveHistoryProof` 除了更新两组 base-37 proof，还会执行：

```text
ComputeCoordinateMix(hx, hy, ax, ay, i)
XorFlagBufferForRound(state, 0xB582A7EB, hx, i, coordinate_mix)
```

所以 proof 只依赖 `(hx+ax, hy+ay)`，flag keystream 却依赖四个具体坐标。随便把坐标和拆成两枚棋子，虽然能命中两个 64 位目标，解出的 39 字节仍是乱码。

每轮在 `0..14` 内拆分坐标和，候选数依次为：

```text
12, 36, 52, 78, 11, 13, 40, 54, 56, 30
```

直接枚举约有 `9×10^14` 种组合。`recover_flag.c` 使用两个关键化简：

1. `native_stage2.c` 把提取出的 PE 映射到首选基址 `0x140000000`，用 GCC 的 Microsoft x64 ABI 直接调用 `ResetProofState`、`ComputeCodeAnchor`、`ComputeCoordinateMix` 和 `XorFlagBufferForRound`。这些函数是确定性的用户态计算，不执行 VMX 指令；
2. flag 的 39 个内层字符都是 ASCII，因此每个明文字节的最高位必为 0。把一轮 39 字节 keystream 的最高位压成一个 39-bit mask，就有：

```text
left_mask XOR right_mask = cipher_high_mask
```

前五轮共有 `19,274,112` 种组合，后五轮共有 `47,174,400` 种组合。把前半的 39-bit mask 放入哈希表，再扫描后半，只会留下大约：

```text
19,274,112 × 47,174,400 / 2^39 ≈ 1.6×10^3
```

个候选对。随后依次检查：

- 39 字节全部位于 `0x21..0x7E`；
- 20 个落点不冲突；
- 双方没有普通五连；
- `ValidatePlaintextSha256` 的内置 32 字节摘要比较通过。

当前实现用两个 `2^26` 项的 `uint64_t` 数组保存前半哈希表，峰值内存约 1 GiB；本机一次求解约 5 秒。若内存较小，可以改为排序后的紧凑记录或把前半表分桶处理。

最后一项的输入可以独立写成：

```python
sha256(
    p32(0xB0D7A925)
    + p32(0xB582A7EB)
    + p32(39)
    + inner_flag
)
```

其中 `p32` 为小端序。正确明文得到：

```text
27cddb503d66bb0cfa3c73e62622831ffae5a92cacc912dec20a9e3f7e54e23f
```

与 `stage2+0x1DB30` 的 32 字节常量完全一致。

在 WSL 中编译并运行：

```powershell
wsl bash -lc 'source /home/kali/miniforge3/etc/profile.d/conda.sh && conda activate ctf-tools && cd "/mnt/d/文档/新建文件夹/D3CTF2026/d3gomoku" && gcc -O3 -march=native -Wall -Wextra -Werror -o recover_flag recover_flag.c && ./recover_flag stage2.dll --ascii; task_status=$?; conda deactivate; exit $task_status'
```

关键输出如下：

```text
MITM combinations: left=19274112, right=47174400
recovered move decomposition:
   1: human=(2,0) ai=(3,1) sum=(5,1)
   2: human=(0,6) ai=(3,2) sum=(3,8)
   3: human=(0,9) ai=(3,3) sum=(3,12)
   4: human=(2,8) ai=(3,4) sum=(5,12)
   5: human=(0,8) ai=(0,10) sum=(0,18)
   6: human=(0,7) ai=(0,5) sum=(0,12)
   7: human=(4,1) ai=(5,2) sum=(9,3)
   8: human=(4,3) ai=(4,2) sum=(8,5)
   9: human=(4,5) ai=(2,2) sum=(6,7)
  10: human=(1,0) ai=(4,4) sum=(5,4)
embedded plaintext SHA-256 check: pass
flag: d3ctf{Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100}
```

完整明文还可以作为 39 字节精确 crib 再跑一次：

```bash
./recover_flag stage2.dll \
  'Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100'
```

两种模式得到完全相同的棋谱，且内置摘要均通过。

证明成功后，`ValidateAndLatchProof` 的真实 flag 链为：

```text
IssueFlagReadToken
  -> ValidatePlaintextSha256 / ValidateFlagStateDigest
  -> CopyDecryptedInnerFlag
  -> WrapInnerFlag
  -> WriteRemoteFlagRegistryValues
```

最后一个函数写入：

```text
HKLM\SYSTEM\CurrentControlSet\Services\drv_game\Parameters
  RemoteFlagStatus
  RemoteFlagReady
  RemoteFlag
```

`RemoteFlag` 就是完整的 `d3ctf{...}` 字符串。

### 6. 只改 AI 选点

第一阶段的人类落子路径位于 `drv_game+0x372C9` 附近。AI 选点调用在：

```text
drv_game+0x3736F  call SelectAiMove
drv_game+0x37374  返回后的第一条指令
```

在 `drv_game+0x37374`：

- `RSI` 是游戏状态；
- `[RSI+0x18CFC0]` 是尚未递增的 AI 历史数量，即轮次 `0..9`；
- `[RSI+0x334]` 是 AI 的 `x`；
- `[RSI+0x338]` 是 AI 的 `y`。

因此不需要改证明函数，也不需要直接把胜负字段写成 0。只在 `SelectAiMove` 返回后覆盖两个坐标，后续棋盘写入、历史记录、滚动哈希、EPT 证明和最终状态写回仍然执行原始代码。

`force_ai.wds` 已写好十个分支。在 WinDbg 中载入：

```text
$$><D:\文档\新建文件夹\D3CTF2026\d3gomoku\force_ai.wds
g
```

脚本使用 `bu` 建立可随模块加载解析的断点，使用 MASM 表达式的 `dwo(...)` 读取轮次，并用 `.if/.elsif` 选择坐标。这些语法可在微软的 [bu 未解析断点说明](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/unresolved-breakpoints---bu-breakpoints-)、[MASM 运算符说明](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/masm-numbers-and-operators) 和 [.elsif 命令说明](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-elsif) 中核对。

### 7. 最后一层 3 bit 门

十轮证明正确还不够。`CheckHiddenHumanWin` 最前面有一个廉价门：

```c
(
    rol32(*(uint32_t *)(ctx + 0x20)
          ^ *(uint32_t *)(ctx + 0x24), 3)
    ^ rolling_hash
) & 7
```

结果必须等于 `3`。每次 reset 使用 QPC 派生新的 `game_id` 和初始滚动哈希，因此相同棋步在不同局里会产生不同结果。这个门只保留 3 bit，单局命中率约为 `1/8`。

没有必要反解整个滚动哈希。`solve_client.py` 每局重放相同十轮证明，失败就 reset，默认最多 64 局。调试用 Unicorn 模型中，连续八个 QPC 初值的第八个样本命中：

```text
QPC = 0x100010007
game_id = 0x0001C356
final rolling hash = 0x91110CFB
final_hash & 7 = 3
```

这组数只用于说明 3 bit 门的行为；真实 VM 中不应依赖固定 QPC。

### 8. Intel Windows 10 虚拟机中的完整运行顺序

动态步骤只应在快照可恢复的测试虚拟机中进行：

1. 使用 Intel 主机；
2. VMware/其他 VMM 向来宾暴露 nested VT-x；
3. 来宾为 Windows 10 19045；
4. VBS/Hyper-V 不占用 VMX；
5. 仅在测试 VM 内信任题目测试证书并加载 `drv_game.sys`；
6. 连接 kernel WinDbg，载入 `force_ai.wds`；
7. 再运行 `solve_client.py`。

典型的驱动服务命令如下，但本文没有在当前 AMD 日常主机上执行这些系统修改：

```powershell
sc.exe create D3Gomoku type= kernel start= demand binPath= "C:\ctf\d3gomoku\drv_game.sys"
sc.exe start D3Gomoku
```

在虚拟机里准备好 Python 后运行：

```powershell
py -3 C:\ctf\d3gomoku\solve_client.py
```

客户端会：

1. 打开 `\\.\D3Gomoku`；
2. 发送固定握手；
3. 启动一个新的 `notepad.exe`；
4. 轮询状态和 reset，等待 VMX 初始化完成；
5. 每轮发送人类落子，并核对 WinDbg 写入后的 AI 坐标；
6. 最后一层哈希未命中时自动重开；
7. 收到 `result=2` 后读取注册表中的 `RemoteFlag` 并输出。

只查看落子计划、不打开驱动：

```powershell
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/文档/新建文件夹/D3CTF2026/d3gomoku/solve_client.py" --print-plan
```

### 9. 第一阶段的 HTTP flag 链为什么是诱饵

第一阶段确实能构造字符串 `d3ctf{`，但交叉引用审计表明它属于一条不可达的 HTTP/JSON 解密链：

```text
BuildProofRequestAndDecryptBlob  drv_game+0x3D413
DecryptBlobAndValidateFlag       drv_game+0x3C7B0
目标地址                          192.168.152.1:8788
```

证据如下：

1. `BuildProofRequestAndDecryptBlob` 没有代码调用者；
2. 它唯一有意义的数据引用在 `drv_game+0x31DE6`；
3. `DriverEntry` 把它与十个同类未引用函数地址 XOR 后，写到 `qword_140148088`；
4. `qword_140148088` 在整个驱动中只有这一次写入，没有任何读取；
5. 第二阶段只扫描棋盘包装器和 verifier 两种签名，不会扫描或调用 HTTP 链；
6. `0x222054` 状态记录只返回遥测字段，没有明文 flag 缓冲区；
7. `192.168.152.1` 是作者环境形态的私网地址，附件没有对应服务。

所以这条链的作用是给“搜索 `d3ctf{` 字符串”制造一个很像答案的死代码目标。把真实 proof target 拼成：

```text
d3ctf{44ea257de1cefb27eed3c641a4c3a7a7}
```

同样没有代码或外部证据支持，不能当作最终 flag。

真正的明文路径完全位于第二阶段，成功时同时满足：

```text
proof_1                 = 0x44EA257DE1CEFB27
proof_2                 = 0xEED3C641A4C3A7A7
ValidatePlaintextSha256 = true
ValidateFlagStateDigest = true
RemoteFlag              = d3ctf{Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100}
```

换言之，第一阶段 HTTP 代码是诱饵，但“附件内没有 flag”这个结论也是错的；必须继续跟到第二阶段的加密状态和注册表写入，才能看到真实可达链。

### 10. 产物说明

| 文件 | 用途 |
| --- | --- |
| `extract_stage2.py` | 从第一阶段提取并解密 `stage2.dll` |
| `stage2.dll` | 解出的 VMX 第二阶段 |
| `proof_solver.py` | 独立反解并验证十轮隐藏证明 |
| `native_stage2.c` | 在 WSL 用户态映射 PE 并调用第二阶段确定性例程 |
| `recover_flag.c` | 39-bit MITM、棋盘检查和内置摘要复核 |
| `recover_flag` | WSL 下编译出的求解器 |
| `choice_keys.bin` | 可选的逐轮候选 keystream 导出表 |
| `solve_client.py` | Windows 用户态 IOCTL 客户端和自动重试 |
| `force_ai.wds` | WinDbg 强制 AI 十步坐标 |
| `solve.py` | Unicorn 研究/指令级验证脚本 |
| `drv_game-annotated.i64` | 第一阶段 IDA 标注数据库 |
| `stage2-annotated.i64` | 第二阶段 IDA 标注数据库 |

`solve.py` 是分析过程使用的重型 harness，需要 `pefile` 和 `unicorn`；`proof_solver.py` 只使用 Python 标准库。最终 flag 恢复只需 GCC、`native_stage2.c`、`recover_flag.c` 和提取出的 `stage2.dll`。

## 方法总结

这题真正的难点有四层。

第一层是环境识别。`drv_game.sys` 不是单阶段棋盘校验，`notepad.exe` 镜像回调会释放一个嵌入 PE，而这个 PE 是 Intel VMX hypervisor。官方的 Intel 提示直接指向决定性主障碍。

第二层是 EPT 视图差异。原始页上的两组常量是诱饵；必须跟进 `VMFUNC` 和影子页生成代码，才能拿到 `0x44EA257DE1CEFB27`、`0xEED3C641A4C3A7A7`。只反编译第一阶段会在这里走错。

第三层是把证明约束与具体坐标分开。先把 64 位目标还原成 base-37 digits，再逐轮解两个模 37 方程，得到唯一坐标和。这里不能“任选”一组分解：逐轮 flag keystream 还依赖四个具体坐标。

第四层是高效恢复具体坐标。把 39 个 ASCII 最高位作为精确 MITM 约束，能把约 `9×10^14` 种组合压到约 1600 个候选，再用完整字符范围、棋盘合法性和附件内置 SHA-256 摘要得到唯一解。动态阶段只改 AI 选点，不跳过证明；最后的 3 bit 门用重开对局解决。最终 flag 由第二阶段自行包装并写入注册表，整个结论不依赖外部 checker。
