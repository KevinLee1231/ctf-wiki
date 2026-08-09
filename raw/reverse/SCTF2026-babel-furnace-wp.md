# babel_furnace

## 题目简述

程序读取恰好 48 字节输入，经 Host、嵌入式 Python 3.11、`bridge.pyd` 和 Rust `engine.dll` 四层组合验证。主程序手动映射 Python 与扩展模块，并在自身保存 320 个、每个 `0x800` 字节的加密 fragment；真正的 VM opcode、操作数、立即数、页密钥、输入置换和目标状态都被拆成多方 share，任何单个模块都不含完整算法。

静态解法不需要模拟整个手写 PE loader。官方 `exp/solve.py` 从发布版 EXE 直接定位 fragment 表，恢复各组件和 Python carrier，再合并 share 还原 80 个 VM block，提取 14 轮、6 个 64 位字的可逆变换并逆运算最终状态。

## 解题过程

### 1. 恢复 Host fragment 链

每条 fragment 包含密文、tag、编码后的下一索引和密钥演化信息。由 bootstrap key 解开首段，校验 tag，再按

```text
next_index = next_encoded XOR mask
next_key   = derive(current_key, fragment)
```

追到 `FLAG_LAST`，即可分别拼回 `bridge.pyd`、`engine.dll`、Python carrier 和公开 final-state blob。用 PE/Machinery header 与声明长度验证每个恢复结果，避免把误命中的 fragment 表继续解析。

### 2. 从 Python code object 提取 share

carrier 的代码对象字段被当作数据容器：

```text
co_exceptiontable -> 80 组 Python token/immediate
co_linetable      -> 16 组 page-key share
co_name           -> block permutation
co_qualname       -> engine 起点、输入置换和 mask share
```

bridge 还提供 Pyd share，Host API 提供 page、edge、target 和 immediate share。恢复代码对象后，应按原运行时的 block permutation 取值，而不是按 carrier 中物理出现顺序顺次读取。

### 3. 解密页并还原 640 条微指令

Rust Engine 共 16 页，每页 5 个 block；每个 block 8 条微指令，总计 640 条。页密钥由四方 share 异或后经 SplitMix 派生。每条逻辑指令同样由多方数据组合，例如：

```text
opcode    = engine_share XOR pyd_token.low5 XOR python_token.low5
immediate = python_immediate XOR pyd_mask XOR engine_mask XOR host_share
```

对页 tag、packet tag 和 opcode 范围逐层验证后，可以去掉 `NOISE`，识别 `LOAD_INPUT`、加法/XOR、旋转、nibble S-box、word permutation、`ASSERT_TAG` 和 `HALT`。

### 4. 提取并逆转核心变换

有效算法把 48 字节解释成 6 个小端 `uint64_t`，执行 14 轮。每轮包含 round-key 加法、相邻字异或、旋转、4-bit S-box、加法混合和 6 字置换。官方脚本从每轮固定语义段提取 add key、rotation 和 permutation，再按完全相反顺序执行逆置换、逆加法、逆 S-box、反向旋转与异或，最终把公开 target 还原为原始 48 字节。

复现：

```bash
python exp/solve.py babel_furnace.exe
```

输出为：

```text
SCTF{1_d0n't_w@nT_tO_crEat3_VMs_AnYmoRe!??!?!!?}
```

## 方法总结

本题的复杂度主要来自“数据被拆到四层”，而不是某条微指令本身。正确策略是先恢复各层权威数据，再以 tag 和数量约束验证 share 合并，最后把 640 条 VM 指令压缩成 14 轮高层变换。直接动态单步会反复穿过手写 loader 和 Python/Rust 边界；官方静态脚本把这些边界变成可测试的解析阶段，且已在本地对发布版 EXE 运行得到上述 48 字节结果。
