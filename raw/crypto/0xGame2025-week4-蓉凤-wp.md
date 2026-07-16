# 蓉凤

## 题目简述

题目又名 AES Square（Integrals）Revenge，目标仍是攻击仅执行 4 轮的 AES-128。与标准 Square 攻击不同，攻击者无法取得包含全部 256 个活动字节取值的完整 $\Delta$ 集，只能获得约 60–70 个互不重复取值组成的子集，即 $\Delta'$ 集。完整集合的“逐字节异或为 0”性质已经消失，需要改用“同一活跃位置的值两两不重复”筛选末轮密钥。

现有仓库和 PDF 没有保留本题附件、交互代码或最终输出；PDF 只指向出题人的原理文章与 C++ 实现。因此下文保留可复现的攻击模型、输入要求和工具用法，不虚构缺失的比赛 flag。

## 解题过程

### 从平衡性质改用不重复性质

普通 Square 攻击构造 256 个明文，使一个字节遍历 `0x00`–`0xff`，其余字节固定。经过 3 轮后，同一状态位置的 256 个值异或为 0；枚举第 4 轮轮密钥并回退一轮，即可用这个平衡条件筛选正确密钥。

本题只有 $\Delta'$ 集，取值没有覆盖整个字节空间，所以异或结果不再固定为 0。但只要样本来自同一个状态、活动字节彼此不同，经过 `SubBytes` 等双射后仍保持两两不同。将密文回退到第 3 轮 `MixColumns` 附近后，正确密钥候选应使对应活跃位置的样本值没有重复。

对一列状态追踪后，第 4 轮 `ShiftRows` 会把相关字节分散到 4 个位置。例如第一组对应 `(0,0)`、`(1,3)`、`(2,2)`、`(3,1)`。这 4 个字节应作为一个 word 联合猜测，而不是逐字节独立判断。

### 为什么可以忽略第 3 轮轮密钥

回退路径概念上为：

```text
AddRoundKey[4]^-1 -> ShiftRows^-1 -> SubBytes^-1
-> AddRoundKey[3]^-1 -> MixColumns^-1
```

第 3 轮 `AddRoundKey` 对同一位置的所有样本只施加相同的固定异或偏移。固定偏移不会改变“值是否重复”，因此验证该性质时可以把第 3 轮轮密钥视为全 0。攻击对象便缩减为第 4 轮同一组的 4 个密钥字节，每组最多枚举 $2^{32}$ 种；4 组相互独立，最终拼出完整第 4 轮轮密钥，再逆转 AES-128 密钥扩展恢复原始密钥。

核心验证逻辑可写成如下伪代码；`group` 已按逆 `ShiftRows` 后的 4 个相关位置分组：

```python
def verify(partial_key, group):
    seen = [set() for _ in range(4)]
    for word in group:
        state = xor_bytes(word, partial_key)
        state = inv_sub_bytes(state)
        state = inv_mix_columns(state)

        for index, value in enumerate(state):
            if value in seen[index]:
                return False
            seen[index].add(value)
    return True
```

该枚举量不适合直接用 Python 跑。作者提供的 [AES-Integrals C++ 实现](https://github.com/CrystalJiang232/AES-Integrals) 使用多线程、预计算和按 word 剪枝；其 README 给出的 66 个密文测试中，预期枚举量约为 $2^{33}$，常见运行时间为数十秒到约两分钟。样本少于 50 个时，假阳性增多，正确率会明显下降。

### 构建与运行 C++ 攻击器

将同一 $\Delta'$ 集的密文按仓库要求保存为十六进制文本，每行一个 16 字节密文。构建后使用 `aes4-2`，默认读取 66 个样本：

```bash
git clone https://github.com/CrystalJiang232/AES-Integrals.git
cd AES-Integrals
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
./build/AES/aes4-2 --it pdelta.txt --count 66 --echo 1
```

如需二进制输入可改用 `--ib`；`--nothread` 会关闭并行并显著增加耗时。程序输出恢复的 AES 密钥，后续再按题目交互使用该密钥。作者的 [完整 $\Delta$ 集分析](https://crystaljiang232.github.io/crypto/aesatk/4-1/) 与 [$\Delta'$ 集 Revenge 分析](https://crystaljiang232.github.io/crypto/aesatk/4-2/) 保留为原理来源和延伸阅读；本题实际依赖的判据与参数已在正文中展开。

## 方法总结

积分攻击的关键不是固定套用“异或和为 0”，而是追踪一组选择明文在中间状态中仍保留的结构。完整 $\Delta$ 集提供平衡性质，部分 $\Delta'$ 集则只能依赖不重复性，代价是把末轮密钥按 4 字节分组联合枚举。识别固定轮密钥偏移不会破坏该性质，可以免去第 3 轮密钥枚举，把攻击压缩到 4 组至多 $2^{32}$ 的并行搜索。
