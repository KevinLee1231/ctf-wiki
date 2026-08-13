# Whats In The Bag

## 题目简述

程序从当前目录打开 `bag.txt`，读取恰好 7 个字节，并按乱序索引检查每个字符。全部检查通过后，它计算这 7 个字符的 ASCII 总和，并把总和与字符串一起嵌入 flag。

## 解题过程

把检查条件按索引重新排序：

```text
contents[0] = 'g'
contents[1] = 'r'
contents[2] = 'e'
contents[3] = 'y'
contents[4] = 'c'
contents[5] = 'a'
contents[6] = 't'
```

所以 `bag.txt` 必须包含 7 字节 `greycat`，不要附加换行。ASCII 和为：

$$
103+114+101+121+99+97+116=751
$$

程序按格式 `grey{th3_b4g_c0nt41ns_%d_%ss!!}` 输出，因此得到：

```text
grey{th3_b4g_c0nt41ns_751_greycats!!}
```

源码中 `char contents[7]` 没有为字符串终止符预留空间，却把它交给 `%s`，严格来说会越界读取直到偶然遇到 `\0`，属于未定义行为。仓库的预期环境能够输出上述结果，但稳健实现应改为 `char contents[8] = {0};`，或使用 `%.7s` 限制输出长度。

字符约束、求和方式和预期输出也见[公开参赛者题解](https://sl-lee.github.io/CTF-Writeups/NUS-Greyhats-Welcome-CTF-2025)；正文已保留不依赖反编译截图的完整推导。

## 方法总结

- 核心技巧：按实际数组索引重排字符约束，并计算固定字符串的 ASCII 校验和。
- 识别信号：多个乱序的 `contents[index] != constant` 分支，以及成功路径中的求和与格式化输出。
- 复用要点：构造输入时注意 `fread` 的精确长度和换行；同时审计 `%s` 所需的 NUL 终止，区分预期解法与源码中的未定义行为。
