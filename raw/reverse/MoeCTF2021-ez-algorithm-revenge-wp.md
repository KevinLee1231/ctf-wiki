# ez_Algorithm_revenge

## 题目简述

程序用固定种子 `114514` 生成一个 50 层数塔，要求输入恰好 49 个字符。每走一步都会进入下一层，其中 `L`、`D`、`R` 分别把列坐标减一、不变、加一；输入内容就是 flag 花括号内的部分。

题目没有在程序里判断分数是否最大，因此需要从生成逻辑恢复相同的数塔，再用动态规划求最大权路径。

## 解题过程

源码中的数据生成逻辑是：

```cpp
srand(114514);
for (int row = 1; row <= 50; row++)
    for (int col = 1; col <= row; col++)
        mapp[row][col] = rand() % 1919810;
```

设 `dp[row][col]` 表示走到第 `row` 层第 `col` 列时的最大累计分数。当前格可能来自上一层的三种位置：

- 从 `col + 1` 走 `L`；
- 从 `col` 走 `D`；
- 从 `col - 1` 走 `R`。

因此，对每个合法前驱取最大值并记录前驱列和对应字符，最后从第 50 层的最大值反向恢复路径即可。

这里有一个容易忽略的实现细节：C/C++ 标准没有规定 `rand()` 的具体序列。题目说明使用的是 Windows 下的 MinGW 工具链，附件程序也使用 Windows C 运行库的随机数实现；如果直接在 Linux/glibc 下编译求解器，即使种子相同也会生成另一张数塔。求解器应在与附件相同的 Windows/MinGW 运行时下编译，或者直接从附件中复现其 `rand()`。

完整求解代码如下：

```cpp
#include <algorithm>
#include <climits>
#include <cstdlib>
#include <iostream>
#include <string>

using namespace std;

constexpr int N = 50;
constexpr long long NEG = -(1LL << 60);

long long mapp[N + 1][N + 1];
long long dp[N + 1][N + 1];
int parent_col[N + 1][N + 1];
char parent_move[N + 1][N + 1];

int main() {
    srand(114514);
    for (int row = 1; row <= N; ++row) {
        for (int col = 1; col <= row; ++col) {
            mapp[row][col] = rand() % 1919810;
        }
    }

    for (int row = 0; row <= N; ++row) {
        for (int col = 0; col <= N; ++col) {
            dp[row][col] = NEG;
        }
    }
    dp[1][1] = mapp[1][1];

    for (int row = 2; row <= N; ++row) {
        for (int col = 1; col <= row; ++col) {
            auto relax = [&](int previous_col, char move) {
                if (previous_col < 1 || previous_col > row - 1) {
                    return;
                }
                long long candidate =
                    dp[row - 1][previous_col] + mapp[row][col];
                if (candidate > dp[row][col]) {
                    dp[row][col] = candidate;
                    parent_col[row][col] = previous_col;
                    parent_move[row][col] = move;
                }
            };

            relax(col + 1, 'L');
            relax(col, 'D');
            relax(col - 1, 'R');
        }
    }

    int col = 1;
    for (int candidate = 2; candidate <= N; ++candidate) {
        if (dp[N][candidate] > dp[N][col]) {
            col = candidate;
        }
    }

    long long best_score = dp[N][col];
    string path;
    for (int row = N; row >= 2; --row) {
        path.push_back(parent_move[row][col]);
        col = parent_col[row][col];
    }
    reverse(path.begin(), path.end());

    cout << best_score << '\n';
    cout << path << '\n';
}
```

在与题目一致的运行时下，程序输出最大分数 `1270732`，以及 49 字节路径：

```text
DDDRDDLRLRDRDRLLRRRDLLLRRLDRRDRRLDDDLDRRDLRDLLRDD
```

把路径放入题目指定的格式，得到：

```text
moectf{DDDRDDLRLRDRDRLLRRRDLLLRRLDRRDRRLDDDLDRRDLRDLLRDD}
```

## 方法总结

这题把逆向得到的数据生成逻辑与经典数塔动态规划组合在了一起。除了写出状态转移和回溯，最重要的检查点是伪随机数实现：固定种子只能保证同一种 `rand()` 实现内的序列可复现，不能跨运行库直接等价。遇到类似题目，应同时记录种子、调用顺序、取模方式和实际运行时。
