# TCL

## 题目简述

题目实现了带对象数组和后台垃圾回收线程的 Tiny Config Language。GC 每 5 秒先记录所有 `refcount == 0` 的全局索引，再逐个释放对象；释放与最终把槽位置 `NULL` 之间还有约 5 ms 间隔。前台解析线程仍可在这个窗口重分配刚释放的块并提高其引用计数。

这会留下两个指向同一堆块的全局引用，结束下一份配置时两个引用都被归零，后续 GC 对同一块执行两次释放，形成可利用的 tcache double free。

## 解题过程

程序启动时直接打印 `alarm` 的地址，先用附件 libc 的符号偏移计算 libc 基址。

第一份配置大量创建同名键和不同整数值，结束后令这些对象进入待回收状态。第二份配置开始后，在 GC 释放阶段按约 5.45 秒加少量抖动发送新的 `legoclones = 1337`。若新对象复用了一个“已 free、尚未置空”的块，其引用计数变为非零；GC 因而保留旧槽位，同时解析器又把新引用追加到对象数组。

再填满两个引用之间的空槽并结束配置，等待下一轮 GC，双重释放同一块。官方脚本随后利用 tcache freelist 污染，把一次整数对象值写成：

```python
target = libc_base + libc.sym["__malloc_hook"] - 0x10
win = elf.sym["win"]

# 官方堆布局中，第 2 个值放 target，第 39 个值放 win。
values[2] = target
values[39] = win
```

后续分配在 `__malloc_hook-0x10` 附近构造假对象，再把 `win` 写入 hook；下一次触发分配即进入 `win` 获得 shell。由于漏洞依赖线程调度和网络抖动，官方脚本以 10 ms 步长枚举约 `5.45 + i*0.01` 秒的发送时机，并通过输出是否出现正常 shell 来判定成功。

最终读取：

```text
byuctf{ok4y_y34h_th4t_d3fin1t3ly_suck3d}
```

## 方法总结

- 核心技巧：在 GC 的 free/null 分离窗口中复用块，制造同一对象的双引用，再将其转化为 tcache double free 和 hook 覆盖。
- 识别信号：并发 GC 若先快照索引、后分阶段 free 与清空，且前台仍能访问或重分配对象，应重点检查 TOCTOU、UAF 和重复引用。
- 复用要点：比赛中的固定索引、延迟和 `__malloc_hook` 都依赖具体 libc 与堆布局；先验证 double free 原语，再针对目标版本选择覆盖点。
