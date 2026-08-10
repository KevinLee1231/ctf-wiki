# Invest in hints

## 题目简述

题目给出 25 条 hint。每条 hint 由一个二进制掩码和一段字符组成：掩码从低位到高位表示 flag 从左到右的位置，掩码中第 $k$ 个为 1 的位置对应 hint 字符串中的第 $k$ 个字符。目标是选择尽可能少的 hint 覆盖 flag 的全部未知位置，并按掩码把字符放回原位。

这实际上是一个规模很小的集合覆盖问题。已知 flag 格式 `hgame{...}` 时，开头六个字符和末尾 `}` 可以视为已经覆盖；是否利用这一格式不会改变信息重组的方法。

## 解题过程

### 1. 把每条 hint 解释为位置集合

设第 $j$ 条 hint 的掩码为 $m_j$。若从最低位开始的第 $i$ 位为 1，则该 hint 覆盖 flag 的位置 $i$。例如，最低位为 1 意味着 hint 的第一个字符属于 flag 的第 0 位。

需要特别注意 `std::bitset` 的显示方向：C++ 使用 `operator<<` 输出时是最高位在左、最低位在右，而题目位置定义从最低位开始。因此读取文本掩码后，应从右向左遍历。

```python
def decode_hint(mask_text, fragment):
    positions = [
        i
        for i, bit in enumerate(reversed(mask_text.strip()))
        if bit == "1"
    ]
    if len(positions) != len(fragment):
        raise ValueError("掩码中 1 的数量与 hint 长度不一致")
    return dict(zip(positions, fragment))
```

用全部 hint 合并字符时，同一位置可能被多条 hint 重复覆盖。重复值应完全一致，否则说明掩码方向或输入解析有误：

```python
def merge_hints(parsed_hints, flag_length):
    flag = [None] * flag_length
    for hint in parsed_hints:
        for position, char in hint.items():
            old = flag[position]
            if old is not None and old != char:
                raise ValueError(
                    f"位置 {position} 冲突：{old!r} != {char!r}"
                )
            flag[position] = char
    return flag
```

### 2. 枚举最少 hint 的覆盖组合

hint 数量只有 $n=25$，总状态数为 $2^{25}=33,554,432$。使用 C++ 枚举所有子集并按位或掩码，在题目规模下完全可行。

下面的程序从 `hints.txt` 读取 `掩码 字符串`，预先把固定 flag 格式标记为已覆盖，然后输出 hint 数量最少的下标集合。`FLAG_LEN` 应设为输入掩码长度。

```cpp
#include <bit>
#include <bitset>
#include <cstdint>
#include <iostream>
#include <limits>
#include <string>
#include <vector>

constexpr int K = 25;
constexpr int FLAG_LEN = 71;

int main() {
    std::vector<std::bitset<FLAG_LEN>> masks(K);
    std::vector<std::string> fragments(K);

    for (int i = 0; i < K; ++i) {
        std::string text;
        std::cin >> text >> fragments[i];
        if (static_cast<int>(text.size()) != FLAG_LEN) {
            return 1;
        }

        // 文本左侧是高位，右侧是最低位。
        for (int j = 0; j < FLAG_LEN; ++j) {
            masks[i][j] = text[FLAG_LEN - 1 - j] == '1';
        }
    }

    int best_count = std::numeric_limits<int>::max();
    std::uint32_t best_subset = 0;

    for (std::uint32_t subset = 0; subset < (1u << K); ++subset) {
        int count = std::popcount(subset);
        if (count >= best_count) {
            continue;
        }

        std::bitset<FLAG_LEN> covered;
        for (int i = 0; i < 6; ++i) {
            covered[i] = true;  // hgame{
        }
        covered[FLAG_LEN - 1] = true;  // }

        for (int i = 0; i < K; ++i) {
            if ((subset >> i) & 1u) {
                covered |= masks[i];
            }
        }

        if (covered.all()) {
            best_count = count;
            best_subset = subset;
        }
    }

    std::cout << "minimum = " << best_count << '\n';
    for (int i = 0; i < K; ++i) {
        if ((best_subset >> i) & 1u) {
            std::cout << i << ' ';
        }
    }
    std::cout << '\n';
}
```

选择出最小覆盖后，把这些 hint 中的字符按第 1 步放回对应位置即可。若只要求恢复 flag 而不要求最少 hint，直接合并全部 25 条会更简单；“最少集合”是题目额外的优化目标。

PDF 中的生成器和标准程序使用的完整结果为：

```text
hgame{Aug5YMkf3o99ACi7Lr0gQSCKaWy2Azq3ti691DhNlCbxu8rR2mCAD5LEwLdmHa42}
```

## 方法总结

本题的障碍是理解掩码方向并重组被分散的字符，而不是密码强度。每条 hint 都可转化为一个位置集合和一组确定字符；25 个集合允许直接枚举所有子集求最小覆盖。实现时最容易出错的是 `std::bitset` 的文本顺序与题目“最低位对应第一个字符”的定义相反，应通过 hint 长度和重复位置一致性做校验。
