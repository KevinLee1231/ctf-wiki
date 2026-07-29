# BFS

## 题目简述

程序伪装成无权图最短路题：输入多组图，程序用 BFS 输出从终点回溯到起点的路径。`n` 被限制在 256 以内，但边端点、查询端点和多组测试之间的全局状态都缺少正确校验。官方利用把数组越界、未清空的 `std::queue<uint8_t>` 与 glibc unsafe unlink 组合起来，最终劫持 C++ 输出函数。

## 解题过程

三个缺陷共同构成利用面。

第一，边端点没有范围检查：

```cpp
std::cin >> from >> dest;
adj_matrix[
    from * MAX_NUMBER_OF_NODES + dest
]++;
adj_matrix[
    dest * MAX_NUMBER_OF_NODES + from
]++;
```

`from` 和 `dest` 是无符号整数，因此可以在 `adj_matrix` 之外对目标字节执行递增。最终路径回溯同样允许 `parent[crawl]` 越界读取。

第二，BFS 找到目标后直接 `return`，没有清空全局队列：

```cpp
q.push(i);
if (i == dest) {
    return;
}
```

libstdc++ 的 deque 底层为 `uint8_t` 队列分配约 `0x200` 字节的数据块；累计推进约 512 个元素会申请下一块，弹出整块后又会释放它。通过连续提交特制测试用例，可以控制这些队列块在 `vis`、`parent` 和 `adj_matrix` 附近的分配与释放时机。

第三，`parent` 数组也没有在每组测试前清零。旧的父节点链会继续参与输出，这既能让程序沿攻击者安排的字节回溯，也能把越界读出的数值打印出来。

官方 unsafe unlink 路线如下：

1. 用前两组图在 `parent` 开头布置伪块的 `fd`、`bk` 等字段；
2. 用多组 256 点图推动 deque，令目标队列块位于可控数组之后；
3. 重复提交越界边，以逐字节递增的方式修改块头：设置伪造的 `prev_size`，扩大 `size` 到 tcache 范围之外，并清除 `PREV_INUSE`；
4. 再推动队列释放该块，触发向后合并和 unsafe unlink；
5. unlink 的双向链表写把 `parent` 相关指针改到 `parent` 自身之前，形成后续指针改写原语。

利用脚本随后通过两轮父节点链，把 `parent` 指向 GOT。每次把目标地址的一个字节安排进回溯链，程序就会以十进制输出该字节。官方脚本从高地址到低地址重组 `alarm@GOT` 的 6 个字节：

```python
libc_base = alarm_leak - 0xea5b0
system = libc_base + 0x50d60
```

最后再次利用 `parent` 写：

- 把 C++ `operator<<` 对应的调用目标改为 `system`；
- 把 `cout` 对象开头改成字节串 `/bin/sh`。

下一次程序执行输出操作时，等价于调用 `system("/bin/sh")`。获得 shell 后读取：

```text
SEKAI{what_do_you_mean_my_integers_have_to_be_checked?_i_never_needed_to_do_that_in_programming_competitions}
```

## 方法总结

本题并非单一的数组越界题。越界递增负责精细修改堆块头，BFS 提前返回负责跨测试保留队列元素并控制 deque 的分配释放，未清空的 `parent` 又负责泄露和后续指针操作。分析算法型 Pwn 时，除了检查索引，还要检查容器和全局数组在多测试用例之间是否恢复到一致状态。
