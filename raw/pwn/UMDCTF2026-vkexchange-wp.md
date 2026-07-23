# vkexchange

## 题目简述

题目把交易数据交给 Vulkan compute shader 处理，并固定使用 Mesa lavapipe 22.3.6。`quote_position` 接受 32768 到 300000 的索引，却把它直接作为只有 1 个元素的 descriptor binding 的 `dstArrayElement`。在关闭 `robustBufferAccess` 后，这会越界写入后续 descriptor set。

目标是利用 lavapipe 的连续主机内存布局，把 settlement shader 的输出缓冲描述符改指向自己的 account 缓冲区，从而逐字读取 oracle flag 缓冲。

## 解题过程

先创建包含 32768 个 outcome 槽位和 4 字节 memo 的市场。服务端按顺序分配 quote、market 和 settlement 三个 descriptor set；固定版本的 lavapipe 使用单调增长的主机 arena，使三者在内存中相邻。

根据该版本布局：

```text
descriptor 大小 = 32
set header      = 88
对齐            = 8
```

因此：

$$
\begin{aligned}
Q&=\operatorname{align}(88+2\cdot32+8,8),\\
M&=\operatorname{align}(88+(32768+1)\cdot32+4,8).
\end{aligned}
$$

令越界索引为

$$
i=\frac{Q+M+32}{32},
$$

则 quote set 的越界 `vkUpdateDescriptorSets` 会落到 settlement set 的 binding 1，也就是输出描述符。

申请一个 256 字节 account 缓冲区，并让越界写生成的 descriptor 指向它。settlement shader 每次把 oracle 缓冲中的一个 32 位字复制到输出缓冲，因此依次提交轮次 $0\ldots63$，每轮审计 account 内容并取回一个 32 位字。拼接所有小端字节并在首个 NUL 处截断，即得：

```text
UMDCTF{yeah_im_sorry_for_making_this_i_know_its_really_annoying_but_at_least_maybe_you_learned_vulkan}
```

## 方法总结

Vulkan 描述符索引错误最终仍是内存破坏，只是内存布局由驱动实现决定。题目固定 Mesa 版本，才使 descriptor 大小、set header 和分配顺序可预测。利用不需要读出 GPU 地址：只需把目标输出描述符重定向到可审计的账户缓冲，就能把 shader 变成逐字拷贝 oracle。
