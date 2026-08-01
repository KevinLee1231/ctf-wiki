# Verse

## 题目简述

Fortnite Verse 代码读取 480 个开关，视为 480 位输入；随后根据一组索引对交换位，最后与一个倒序保存的目标位数组比较。目标是反演这组置换并把位串还原为 60 字节文本。

## 解题过程

从 `verse.verse` 提取目标布尔数组与 240 对索引。代码比较前先对目标顺序做反转，因此先执行：

```python
desired = target_bits[::-1]
```

每个索引只出现在一个 pair 中，整个变换是若干互不相交的 swap。交换是自身的逆运算，所以把目标位按同样 pair 再交换一次，就得到玩家应设置的原始开关：

```python
input_bits = [None] * 480
for x, y in pairs:
    input_bits[x] = desired[y]
    input_bits[y] = desired[x]

bit_string = "".join("1" if b else "0" for b in input_bits)
flag = "".join(
    chr(int(bit_string[i:i + 8], 2))
    for i in range(0, len(bit_string), 8)
)
print(flag)
```

按每 8 位大端组合字符，得到：

```text
byuctf{this_language_is_supposed_to_be_beginner-friendly???}
```

## 方法总结

大量开关与混淆变量最终只是一个比特置换。先明确目标数组是否反转，再建立输出索引到输入索引的映射；对互不相交的 swap，逆置换就是原置换本身，无需暴力枚举 $2^{480}$ 种输入。
