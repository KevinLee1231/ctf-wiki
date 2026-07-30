# NepCTF2026 David Die Binary Writeup

## 题目简述

附件是同时包含 arm64 与 x86_64 slice 的 Fat Mach-O。两个架构并非重复实现：arm64 负责解密第二阶段、恢复偶数轮密钥并驱动会话，x86_64 侧运行自定义 VM、提供奇数轮密钥并校验 IPC 数据包。最终还原全部 15 个独立轮密钥后，才能解开 `XENC` 容器中的两块密文。

底层轮函数借用了 AES 的 S-box、ShiftRows 和 MixColumns，但每个轮密钥独立生成，AddRoundKey 还依赖当前状态与轮号，因此不能套用标准 AES 密钥扩展或加解密接口。

## 解题过程

先分别抽取两个 Mach-O slice，并梳理跨架构职责：

```text
arm64 slice                         x86_64 slice
-----------                         ------------
解密 stage-2 并派生 MASK           解密 VM bytecode
从 overlap 恢复 8 个偶数轮密钥     VM 输出 7 个奇数轮密钥
驱动 packet session                校验 packet 并运行奇数轮
解析 XENC、验证候选                维护 helper state
```

### 恢复 x86_64 的奇数轮密钥

x86_64 自定义数据区中的 VM blob 使用该 slice 的 UUID 和 16 字节 `CTAB_X` 加密：

```text
k_x86 = uuid_x[0:4] || CTAB_X[0:4] || uuid_x[8:12] || CTAB_X[4:8]
code[i] = blob[i] XOR k_x86[i mod 16] XOR ((0x9b * i) mod 256)
```

VM 只有 `LDI`、`XORI`、`ROL`、`SBOX`、`EMIT`、`NOP` 等简单操作。顺序解释字节码，每遇到 `EMIT` 记录累加器，最终得到 112 字节：

```text
RK1 || RK3 || RK5 || RK7 || RK9 || RK11 || RK13
```

### 恢复 arm64 的偶数轮密钥

arm64 入口附近存在 file offset 重叠的自定义 segment，其中还混入了假候选。枚举 overlap 头部并按下式解密，只有结果以 `MH_MAGIC_64` 开头的候选才是真正的 stage-2：

```text
k_arm = uuid_a[0:4] || CTAB_A[0:4] || head[0:4] || CTAB_A[4:8]
stage2[i] = stage2_enc[i] XOR k_arm[i mod 16] XOR ((0x37 * i) mod 256)
```

解析第二阶段 Mach-O，从自定义 16 字节 section 取 `seed`，再对 `__TEXT,__text` 计算摘要，得到：

```text
MASK = PRF4(seed, SBOX_hash(stage2.__text))
```

overlap 中紧跟头部的 128 字节与重复的 `MASK` 异或，恢复：

```text
RK0 || RK2 || RK4 || RK6 || RK8 || RK10 || RK12 || RK14
```

### 重放状态并解密 XENC

arm64 与 worker 的 IPC 会先交换 `nonce_a`、`nonce_w` 和 `helper_tag`，随后派生会话密钥和 transcript：

```text
session_key = SPONGE(DOM_SK, PSK || nonce_a || nonce_w)
transcript  = SPONGE(DOM_TH, nonce_a || nonce_w)
```

奇数轮请求结构为：

```text
seq[4] || round[1] || state[16] || MAC[8]
```

其中：

```text
MAC = SPONGE(DOM_MAC, session_key || seq || round || state)[0:8]
```

nonce 与会话密钥只用于保护 IPC 返回掩码；`helper_state` 从 `PSK || helper_tag` 派生，后续更新完全由已知请求和真实轮输出决定。因此拿到 15 个轮密钥后，可以离线重放整个状态机，不必真实启动双架构会话。

`MASK` 的另一个使用点可解出 `XENC` 容器：

```text
g_ovl[i] = XENC_DATA[i] XOR MASK[i mod 16]
```

容器结构为：

```text
"XENC" || version || algorithm || parts || key_len ||
iv_len || data_len || IV || ciphertext
```

这里 `iv_len = 16`、`data_len = 32`。自定义 AddRoundKey 对进入本轮前的状态快照 `t` 进行反馈：

```text
d[j] = rol8(t[(j + r) mod 16], (r + 1) mod 8)
       XOR t[(5 * j + 1) mod 16]
state[j] = state[j] XOR rk[j] XOR d[j]
```

对两个 counter 分别运行 14 轮，所得 16 字节输出即 keystream，与两块 ciphertext 异或后得到 inner flag，最后按 `NepCTF{<inner>}` 组装结果。

## 方法总结

本题的难点在于信息被故意分散到两个架构、伪 overlap、第二阶段 Mach-O、VM 与 IPC 状态机中。可靠做法是先建立数据依赖图，再逐层恢复：x86 VM 给奇数轮密钥，arm64 stage-2 给 MASK 和偶数轮密钥，最后才重放轮函数。需要特别避免两个误区：一是把 15 个独立轮密钥误当成 AES 主密钥扩展，二是忽略 state-dependent AddRoundKey 而直接调用标准 AES 实现。
