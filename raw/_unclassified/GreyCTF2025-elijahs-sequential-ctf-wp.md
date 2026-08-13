# Elijah's Sequential CTF

## 题目简述

这是一道交互提交式的算法题。给定长度为 $n$ 的类别序列，其中 $2\le n\le10^6$、每项 $a_i\in\{0,1,2\}$；可以按原顺序选择任意子序列。相邻已选类别发生变化时，根据方向获得不同满意度：

```text
0 -> 1: +2    1 -> 2: +5    0 -> 2: +3
1 -> 0: +4    2 -> 0: +1    2 -> 1: +6
```

目标是最大化总满意度。题面 PDF 共两页：第一页给出完整定义、约束、两组样例与解释；第二页仅补充子序列定义及提交命令。两页已逐页对照检查，均为纯文本信息，因此正文转写关键内容，不保留页面截图。

## 解题过程

令 `best[c]` 表示扫描到当前位置时，所有“以类别 `c` 结尾的已选子序列”可取得的最大满意度。遇到当前类别 `c` 时，新的最优解有三种来源：跳过当前元素、从空子序列开始、或把当前元素接到另一类别 `p` 后面：

$$
\operatorname{best}'[c]=\max\left(\operatorname{best}[c],0,\max_{p\ne c}(\operatorname{best}[p]+w_{p,c})\right).
$$

另外两个类别的状态保持不变。由于转移只依赖三个历史最大值，不需要保存长度为 $n$ 的 DP 数组。下面是与官方 `simpler.cpp` 等价的实现：

```cpp
#include <algorithm>
#include <array>
#include <iostream>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n;
    cin >> n;
    const int NEG = -1000000000;
    array<int, 3> best{NEG, NEG, NEG};

    for (int i = 0; i < n; ++i) {
        int c;
        cin >> c;
        if (c == 0) {
            best[0] = max({best[0], best[1] + 4, best[2] + 1, 0});
        } else if (c == 1) {
            best[1] = max({best[1], best[0] + 2, best[2] + 6, 0});
        } else {
            best[2] = max({best[2], best[0] + 3, best[1] + 5, 0});
        }
    }

    cout << max({best[0], best[1], best[2]}) << '\n';
}
```

算法时间复杂度为 $O(n)$，额外空间为 $O(1)$。用题目仓库保存的 10 组测试数据逐一编译运行，输出全部匹配；提交后服务返回：

```text
grey{1m_s0m3whaT_oF_4_c0mp3tit1vE_pR0gramm3R_mYsELF}
```

## 方法总结

子序列最优化不一定需要按位置保存完整 DP。只要未来收益只取决于最后一个已选类别，就可把历史压缩成每种结尾状态的最优值。本题转移具有方向性，`0 -> 1` 与 `1 -> 0` 的分数不同，不能把边权当成无向；同时保留原状态即代表跳过当前题目。
