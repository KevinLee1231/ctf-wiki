# abyssgate

## 题目简述

题目由用户态 loader、第二阶段 ELF、eBPF tracepoint 和内核模块 `abyss.ko` 共同完成校验。用户态程序会解包出第二阶段 ELF，第二阶段再通过 ioctl 与内核模块交互；eBPF 在 ioctl 进入和返回时改写参数，因此必须把三部分数据流合在一起分析。

## 解题过程

整体执行链为：

```text
userspace loader
  -> second-stage ELF
  -> eBPF sys_enter_ioctl / sys_exit_ioctl
  -> abyss.ko
```

外层 `abyssgate` 是静态链接、stripped ELF，字符串中能看到：

```text
TracerPid:
/proc/%d/comm
sudo
LD_PRELOAD
```

说明外壳带有反调试、反注入检查。运行时它会解密内嵌 blob，写入匿名 `memfd`，再通过 `execveat` 执行：

```text
memfd_create(...)
write(memfd, payload, ...)
execveat(memfd, "", argv, envp, AT_EMPTY_PATH)
```

真正与 `/dev/abyss` 和 eBPF 交互的是第二阶段 ELF。提取方式可以是从运行中的 `/proc/<pid>/exe` 复制，也可以拦截 `memfd_create`/`write`/`execveat` 导出 payload。第二阶段字符串包括：

```text
Enter the flag:
/dev/abyss
syscalls/sys_enter_ioctl
syscalls/sys_exit_ioctl
```

ioctl 顺序为：

```text
0xAB00       init
0xC018AB04   negotiate
0x4018AB05   commit
0xAB02       finalize
0x4024AB03   check
```

完整流程：

```text
ioctl(fd, 0xAB00)
for i in range(4):
    ioctl(fd, 0xC018AB04, negotiate)
    ioctl(fd, 0x4018AB05, commit)
ioctl(fd, 0xAB02)
ioctl(fd, 0x4024AB03, check)
```

关键结构体为：

```c
struct abyss_negotiate {
    uint32_t block_idx;
    uint32_t nonce;
    uint8_t  challenge[8];
    uint32_t ko_token;
    uint32_t reserved;
};

struct abyss_commit {
    uint32_t block_idx;
    uint8_t  data[8];
    uint32_t token;
    uint8_t  tweak[8];
};

struct abyss_check {
    uint8_t expected[32];
    int32_t result;
};
```

输入被拆成 4 个 8-byte block。eBPF 的作用不是观察，而是主动改写参数：

```text
NEGOTIATE enter:
  修改 nonce，内核看到的不是用户态原始值。

NEGOTIATE exit:
  challenge 按字节异或固定 mask；
  ko_token 的四个字节分别过 S-box。

COMMIT enter:
  data 经过 4 轮 SPN；
  token 经过掩码和旋转；
  tweak 经过 S-box + XOR；
  block_idx 被置换为 2,0,3,1。
```

内核模块 `abyss.ko` 保留了较多符号，例如：

```text
abyss_negotiate
abyss_commit
abyss_check
handle_negotiate
handle_commit
handle_finalize
```

`NEGOTIATE` 阶段根据 eBPF 改写后的 `nonce`、当前 `accumulator`、轮密钥和块索引生成 `challenge` 与 `ko_token`。返回用户态后，这两个值又会被 eBPF 修改，用户态后续派生 `token` 和 `tweak` 时使用的是修改后的响应。

`COMMIT` 阶段会：

```text
1. 检查协议状态；
2. 对上一块密文做 cross-block feedback，更新轮密钥；
3. 将 data ^ tweak 作为本轮输入；
4. 使用 8 轮 64-bit Feistel/Lai-Massey 结构加密当前 block；
5. 用本轮密文和 token 更新全局 accumulator。
```

4 个 block 处理完后，`FINALIZE` 对 32 字节结果做两轮扩散：

```text
forward diffusion
reverse diffusion
```

最终 `CHECK` 将 `final_result` 与用户态给出的 `expected[32]` 常量时间比较。

求解顺序应与执行链相反：

```text
1. 逆转 FINALIZE，恢复 32 字节块密码输出。
2. 逐块重建协议参数：
   原始 nonce
   eBPF 改写后的 nonce
   内核 challenge / ko_token
   eBPF 改写后的 challenge / ko_token
   用户态 token / tweak
   eBPF 改写后的 token / tweak
3. 逐块逆转内核 8 轮 Feistel/Lai-Massey，并去除 tweak。
4. 逆转 eBPF commit 阶段的 4 轮 SPN。
5. 逆转第二阶段用户态最开始对 8-byte block 做的扰动。
```

重建求解脚本时需要保留的核心常量如下。`CHECK` 中传入的 32 字节目标密文来自第二阶段的 `expected.h`：

```python
expected_ciphertext = bytes.fromhex(
    "a58353a9c2c24b5f7eb82b77e35c9f4d"
    "e38da70dce9596137f480a81f9968718"
)
```

用户态最开始对每个 8 字节 block 的扰动使用固定 key 和 rotate 表：

```python
scramble_keys = [
    [0x3c, 0xa7, 0x51, 0xe9, 0x0f, 0x8b, 0xd2, 0x64],
    [0x7e, 0x19, 0xf3, 0x45, 0xac, 0x6d, 0x28, 0xb0],
    [0xd1, 0x4a, 0x86, 0x2f, 0xe7, 0x53, 0x9c, 0x01],
    [0x5b, 0xc8, 0x34, 0x7d, 0x60, 0xaf, 0xe2, 0x16],
]

scramble_rots = [
    [3, 5, 1, 7, 2, 6, 4, 0],
    [1, 4, 6, 2, 5, 0, 3, 7],
    [6, 2, 5, 0, 7, 3, 1, 4],
    [4, 7, 3, 6, 1, 5, 0, 2],
]
```

其余 eBPF/内核密码参数都由固定字符串经 SHA-256 生成，至少包括：

```text
AbyssGate_Ko_SBox_2026
AbyssGate_eBPF_SBox_2026
AbyssGate_eBPF_RoundKeys_2026
AbyssGate_eBPF_Perm_2026
AbyssGate_eBPF_ExitSBox_2026
AbyssGate_eBPF_NonceKeys_2026
AbyssGate_eBPF_ExitMask_2026
AbyssGate_eBPF_TokenMask_2026
AbyssGate_eBPF_TweakKeys_2026
AbyssGate_Ko_ChallengeSalt_2026
AbyssGate_AccumulatorInit_2026
AbyssGate_FK_2026
AbyssGate_CK_0..7
AbyssGate_MasterKey_ACTF2026
```

`COMMIT` 阶段的块索引置换为：

```python
IDX_PERM = [2, 0, 3, 1]
```

用户态构造每组 `data` 的逻辑虽然混淆，但有逐字节三角性质：当前 hex 位只会影响当前字节及其后续字节，不会反向影响前面的字节。因此可以把 `abyss.ko` 链接到用户态，patch 掉环境检测函数，用 ptrace 模拟第二阶段的 eBPF/open/ioctl，让真实 `cmd4/cmd5` 和 eBPF 参数变换参与计算；每组从左到右枚举 16 个 hex 字符，只比较当前前缀字节是否等于目标 data。四组恢复后，用完整模拟流程验证 `check` 成功。

最终得到：

```text
ACTF{b02f97ee296b1218c4b771e3cbc21120}
```

## 方法总结

- 核心技巧：把第二阶段 ELF、eBPF 参数改写和内核模块协议状态机合并建模，再按执行链反向求解。
- 识别信号：用户态 ioctl 参数和内核中看到的值不一致，且存在 `sys_enter_ioctl`/`sys_exit_ioctl` tracepoint 时，应优先检查 eBPF 是否主动改写用户缓冲区。
- 复用要点：跨层校验题不要只逆用户态或只逆内核；协议参数在 enter/exit 两侧可能各被改写一次，必须记录每个阶段实际使用的值。
