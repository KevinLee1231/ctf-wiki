# SUCTF2026-Revird

## 题目简述

附件包含 `chal.exe` 和 `Revird.sys`。`chal.exe` 的可见 `main` 只做 48 字节异或比较，得到的是假 flag；真正入口位于函数表中 `main` 之前。真实链路先从外层程序解出一个 0x4410 字节的 worker PE，再通过 Process Hollowing 将其装入挂起的子进程。worker 读取用户输入，打开设备 `\\.\Revird`，通过 `DeviceIoControl` 与驱动协同完成一个魔改 AES-CBC，并把 64 字节结果与内置密文比较。

本题的关键不是单独逆向 EXE 或驱动，而是把 R3 worker 与 R0 驱动各自承担的轮函数重新拼成一个完整、可逆的分组密码。

## 解题过程

### 1. 识别假校验并提取 worker

可见 `main` 要求输入长度为 48，再逐字节检查：

```c
if ((input[i] ^ 0x5a) == decoy[i])
    ...
```

这条分支只能得到假 flag。继续查看 `main` 前一个函数，可以发现另一套完整输入和校验逻辑。它先执行一段魔改 AES 解密循环；在循环结束处观察输出缓冲区，可以看到 `MZ` 文件头，长度为 `0x4410`。因此可在调试器中直接转储该缓冲区，或在 IDA 中按实际加载地址导出：

```python
from idaapi import get_byte

addr = 0x153C7717880  # 以本次调试的实际地址为准
data = bytes(get_byte(addr + i) for i in range(0x4410))
open("worker.exe", "wb").write(data)
```

后续代码使用哈希解析 API，并执行标准的 Process Hollowing：

1. 以自身路径创建挂起进程，并用管道把输入传给子进程标准输入；
2. 读取子进程 PEB 中的原始 ImageBase；
3. 调用 `NtUnmapViewOfSection` 卸载原映像；
4. 在子进程中分配空间，修复 worker 的重定位并写入 headers、sections；
5. 回写 PEB 的 ImageBase，修改线程上下文使入口指向 worker；
6. 恢复线程，等待结束并根据退出码判断成功或失败。

外层还调用 `CheckRemoteDebuggerPresent`，结果会影响下一次 API 哈希。动态提取时应只修正这一环境依赖值，不能跳过后续 worker 逻辑。

### 2. 恢复 worker 与驱动的通信协议

worker 从标准输入读取字符串后打开设备：

```c
CreateFileA("\\\\.\\Revird", GENERIC_READ | GENERIC_WRITE,
            0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
```

随后它以 IOCTL `0x222000` 和驱动通信。输入包为 0x40 字节，关键字段可整理为：

```c
struct R0Packet {
    uint32_t magic;          // 内存字节为 "IVER"，源码多字符常量写作 'REVI'
    uint32_t exception_code;
    uint32_t opcode;
    uint32_t round;
    uint8_t  state[16];
    uint8_t  data[16];
    // 其余辅助字段
};
```

worker 用自定义异常、`int 3`、除零和访问空地址等不同异常进入统一异常处理器，再由处理器构造包调用驱动。驱动检查 magic 和 IOCTL 后按 `opcode` 分发：

| opcode | 驱动侧实际作用 |
|---:|---|
| 5 | `state ^= driver_round_key[0]` |
| 2 | 生成掩码并完成驱动侧查表，是合成 S-box 的一部分 |
| 3 | 一次 `ShiftRows` |
| 4 | `MixColumns`，再异或当前轮的 driver round key |
| 6 | `state ^= driver_round_key[10]` |

worker 最终取得 64 字节加密结果，与映像 RVA `0x43c0` 处的 64 字节常量比较。

### 3. 化简 opcode 2

`opcode 2` 的数据构造看起来复杂，但用户态发包前已经令：

```text
data0 = state
data1 = state ^ G
```

驱动再次异或相同的 `G`：

```text
data1[i] ^ G[i] = state[i]
driver_out[i] = table_t[state[i]]
```

worker 收包后又执行：

```text
new_state[i] = driver_out[i] ^ rand_table[state[i]]
```

所以整条跨权限链可以化为一个纯字节置换：

```text
S[x] = table_t[x] ^ rand_table[x]
```

其中 `table_t` 位于驱动映像 RVA `0x3360`，长度 256；`rand_table` 由固定 LCG 生成：

```python
seed = 0xC0FFEE13
rand_table = []
for _ in range(256):
    seed = (seed * 0x19660D + 0x3C6EF35F) & 0xffffffff
    rand_table.append((seed >> 24) & 0xff)

sbox = [table_t[i] ^ rand_table[i] for i in range(256)]
inv_sbox = [0] * 256
for x, y in enumerate(sbox):
    inv_sbox[y] = x
```

### 4. 合并 R3/R0 的魔改 AES

worker 和驱动各保存一把 16 字节 key，并分别做相同的密钥扩展。两侧的 AddRoundKey 连续执行，因此可预先合并：

```text
effective_round_key[r] = worker_round_key[r] ^ driver_round_key[r]
```

从映像提取所需材料：

```python
worker_sbox = worker_img[0x42B0:0x43B0]
rcon       = worker_img[0x43B0:0x43C0]
target     = worker_img[0x43C0:0x4400]  # 64 字节 CBC 密文
worker_key = worker_img[0x4400:0x4410]
iv         = worker_img[0x4410:0x4420]

driver_key = driver_img[0x3348:0x3358]
table_t    = driver_img[0x3360:0x3460]
```

官方样本中两把 key 和 IV 分别为：

```text
worker_key = 114a2c907f3d58e4b26a0dc53389f120
driver_key = 63a941270ed4961258c0b37a9d1624e8
iv         = 9b2847d16ea5308cf21459c37a0de861
```

完整单块加密流程为：

```text
state ^= effective_round_key[0]

for round = 1..9:
    state = SBox(state)             # R0 与 R3 合成的 opcode 2
    state = ShiftRows(state)        # 驱动 opcode 3
    state = ShiftRows(state)        # worker 本地操作
    state = MixColumns(state)       # 驱动 opcode 4
    state ^= effective_round_key[round]

final round:
    state = SBox(state)
    state = ShiftRows(state)
    state = ShiftRows(state)
    state ^= effective_round_key[10]
```

与标准 AES 相比有三个决定性差异：S-box 是两份表的异或合成；每轮执行两次 `ShiftRows`；轮密钥由 R3 与 R0 两套扩展结果异或得到。连续两次 `ShiftRows` 在本题的状态布局下是自逆置换，因此解密时仍执行两次即可。

### 5. 逆 CBC 得到 flag

实现逆 S-box、逆 MixColumns 和逆轮顺序后，按 CBC 规则逐块解密：

```python
plaintext = bytearray()
prev = iv

for off in range(0, len(target), 16):
    c = target[off:off + 16]
    p = decrypt_custom_aes_block(c, effective_round_keys, inv_sbox)
    plaintext.extend(a ^ b for a, b in zip(p, prev))
    prev = c

pad = plaintext[-1]
assert 1 <= pad <= 16
assert plaintext[-pad:] == bytes([pad]) * pad
flag = bytes(plaintext[:-pad])
```

逆单块的顺序为：先异或第 10 轮 key、两次 `ShiftRows`、逆 S-box；随后从第 9 轮倒序执行 AddRoundKey、逆 MixColumns、两次 `ShiftRows`、逆 S-box；最后异或第 0 轮 key。

使用官方 exp 对附件中的常量复算，输出为：

```text
SUCTF{D0_y0U_unD3r5t4nd_Th15_m491c4l_435?_41218}
```

## 方法总结

- 遇到明显过短、与其它代码规模不匹配的入口校验，应继续检查启动函数、函数表和隐藏调用链；本题可见 `main` 是假校验。
- Process Hollowing 只是第二阶段的装载手段。转储出 worker 后应把分析重心转到设备对象、IOCTL、包结构和驱动分支。
- 跨 R3/R0 的操作要按数据流合并：两套轮密钥可异或，`opcode 2` 可化为单个 S-box，两次 `ShiftRows` 必须同时保留。
- 最终密文采用 CBC 且带 PKCS#7 padding；单块逆变换正确后，还要异或前一密文块或 IV，并验证 padding，才能把局部实现错误与正确明文区分开。
