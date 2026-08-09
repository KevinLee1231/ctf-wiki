# Zombie_progression

## 题目简述

程序展示一个 6×6 魔方并读取转动序列。表面目标像是复原魔方，实际验收还绑定输入文本的 parser digest、36 步提交、多个进程/线程间的 IPC 轨迹、fd/TLS 状态、broker MAC、capability 和逐步 receipt。即使另一串转动在几何上得到相同魔方，也会因文本或隐藏状态不同而失败。

官方二进制内嵌 12 个加密 block capsule，每个块对应最多 3 步。官方 `exp/solve.py` 静态恢复初始状态与 capsule，逐块枚举语法允许的 move token，并用每步隐藏状态与块末认证值剪枝，得到唯一的 36 步 canonical sequence。

## 解题过程

### 1. 不把可见魔方当作全部状态

解析器最大允许 512 步，但生成常量明确要求 36 个 commitment。执行期间，face/line/slice worker、dispatcher、broker、watchdog 和 validator 通过 IPC 更新多组隐藏摘要。最终 flag 的 ChaCha20 材料不仅依赖可见 cube，还依赖：

```text
step receipt
path digest
capability chain
anchor digest
distributed route digest
TLS mesh digest
peer proof / broker auth
```

所以直接调用通用 6×6 solver，即使输出等价复原序列，也不会满足隐藏校验。

### 2. 解开 12 个 block capsule

脚本先在 stripped ELF 中定位唯一 capsule 前缀和初始状态结构。第 0 块使用 bootstrap context 派生 ChaCha20 key；之后每块 key 都由上一块 receipt、path/capability/anchor/route/TLS 等状态链接派生。解密后可以取得该块的 move count、expected receipt、broker auth 和其他认证目标。

这条链必须顺序处理：前一块尚未确定时，后一块的密钥上下文也未知，不能把 12 块独立并行爆破。

### 3. 每块枚举最多三步

从程序实际支持的 move token 集合出发，对当前块的第 1–3 个位置逐层扩展候选。每加入一个 token，就用离线模拟器更新 cube 和全部隐藏状态；不满足中间 MAC 的分支立即丢弃。块末再比较：

```text
broker_auth == capsule.broker_auth
last_step_receipt == capsule.expected_step_receipt
```

每块都只剩一个分支，随后把完成状态提升为下一块 context。恢复出的序列为：

```text
3Rw' U2 F 2L' z Rw2 B 3U x' 2R Fw U' 3Dw2 L B2 y 2F' Rw
D2 3Lw' z2 U F' 2B R2 x 3Uw' L2 Dw Fw' 3R U2 B' y' 2D 3Fw
```

### 4. 用原二进制验证

执行：

```bash
python exp/solve.py Zombie_progression --run
```

脚本先检查发布二进制 SHA-256，再逐块打印 receipt，最后把序列送回原 ELF。本地实跑原二进制得到：

```text
SCTF{Qy@S_1s_R1ght_14332516_@_114514?!}
```

## 方法总结

这道题刻意让“可见魔方状态”成为不完整模型。决定正确性的其实是文本解析、IPC 路径和认证状态共同构成的扩展状态机。12 个 capsule 把原本约 $117^{36}$ 的不可行搜索切成 12 个最多三步的小搜索，并用 receipt/MAC 提供强剪枝。最终序列已经通过官方 SHA-256 对应的原 ELF 实跑验证，因此不仅是静态脚本自洽的候选。
