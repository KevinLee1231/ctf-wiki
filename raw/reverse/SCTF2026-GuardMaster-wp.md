# GuardMaster

## 题目简述

附件是 Android APK。公开 DEX 只负责派生 64 字节 loader key/seed，解析 `assets/guard.gmx` 的 GMX1 容器并恢复隐藏 DEX；真正入口在隐藏类 `gm.core.Dispatcher.check`，它把候选 flag 经 77 步 Java 状态变换组装成 184 字节 packet，再交给 native `finalizeCheck` 扩展为 192 字节并与加密 target 比较。

整个校验链是可逆的。解法不是猜输入，而是从 native target 逆回 final packet，取出 Java 最终状态 `f8`，按相反顺序撤销 77 步调度，再正向重放所有层验证。

## 解题过程

### 1. 解开 GMX1 容器

从公开 Java loader 恢复 key、seed、字符串表和 record descriptor。GMX1 中不同 record 使用的 transform 不同，包括：

```text
XOR
white-box byte transform
SM4-prefix（只处理记录前缀）
RC4
zlib/Inflater
```

每条记录先按 descriptor 与 KDF 派生独立材料，再按记录类型逆变换。解出 manifest 后按其中的 offset、长度和 hash 拼回 DEX front，与其余片段合并，校验 DEX magic/checksum，得到隐藏 DEX。只对整个容器盲跑单一算法会失败，因为 SM4 记录甚至只加密前缀。

### 2. 还原 Java 调度与 native 接口

隐藏入口 `gm.core.Dispatcher.check` 先规范化用户字符串，然后维护多个 64 字节状态；混淆 wrapper 最终映射到少量核心操作。对 77 行 f8 调度逐条记录“输入状态、参数、输出状态”，归并出可逆操作，重点包括：

```text
m11_state_mix
m13_state_permute
m40_token_round_mix 前的 28 轮变换
```

Dispatcher 最终把 f8、派生 token、校验字段等装入 184 字节 packet。native 不是直接接收原 flag，因此若只在 JNI 返回值处 patch 成 true，无法恢复提交所需字符串。

### 3. 逆转 native finalizeCheck

`finalizeCheck` 将 packet 补成三个 64 字节块，对每块执行由静态常量驱动的字节置换、异或/加法、Feistel 和 white-box 层，再与内置 target 比较。逐层写出逆函数：置换用逆置换表，模 $2^8$/$2^{64}$ 加法用减法，Feistel 按轮逆序，查表层用唯一逆表。

从 target 逆运算得到唯一 192 字节 expanded packet，检查补位和 packet 固定字段后截回 184 字节。由 packet 布局提取最终 64 字节 f8 以及调度所需的伴随状态。

### 4. 逆转 77 步 Java 状态

按 Dispatcher 调用顺序倒序执行各操作的逆函数，恢复规范化后的 64 字节用户输入。随后必须做完整正向验证：

1. 重新执行 77 步，确认 f8 与逆出的 final packet 一致；
2. 重新组装 184 字节 packet；
3. 执行 native 正向变换，确认 192 字节结果逐字节等于 target；
4. 用 APK 的 normalization 处理候选 flag，确认不会发生二次改写。

最终得到：

```text
SCTF{2^i-BBv#Hk@v,sVk7e+)sApDf452_cRox1iupJhKoRzQy_RLu_TE0-n%gn}
```

源目录 `README.md` 中出现的 `SCTF{You_can_solve_this?...}` 属于错误复用的其他题 flag，与 GuardMaster 的 64 字节状态、packet 和正向校验均不一致；这里采用的是官方详细 Writeup 第 703 行结果，并由完整逆/正向链约束。

## 方法总结

本题应按“容器 → 隐藏 DEX → Java 状态机 → native packet”分层处理。每层先建立格式和 hash 检查，再进入下一层，可以防止错误 key 在深层才表现为乱码。面对可逆校验，最可靠的方法是从常量 target 逆推并正向回放；仓库中的 README flag 不能因为位置显眼就被视为权威，能通过实际校验链的结果才是证据。
