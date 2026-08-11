# sheep farm simulator

## 题目简述

牧场对象只有 `SHEEP_MAX=20` 个指针，但买入、升级、出售和查看都只拒绝 `idx >= 20`，没有拒绝负数。负索引可越界读写 `game_t` 前方内存。再加上 upgrade 的校验 `if (upgrade_type != 1 || upgrade_type != 2)` 恒真却只打印错误、不终止，攻击者可反复执行加 1 和乘 2，把 `wps` 当作逐位可写的 64 位槽位。

利用链以负索引进入 freed tcache entry，结合 safe-linking 泄露 heap、伪造 tcache fd 建立任意写，进而泄露 libc/PIE 并将 sheep ability table 改为 `system`。

## 解题过程

### 负索引与可控数值写入

先买入低价 sheep 积累 wool，再出售其 chunk。查看 `index=-69` 会把 tcache safe-linked fd 作为 `wps` 打印；其高位恢复出 heap base。对同一负索引反复升级 2 可把 `wps` 归零，再按目标值的二进制位交替执行乘 2 与加 1，写出任意 64 位整数。

官方 solver 使用 safe-linking 编码：

```python
encoded_fd = target ^ (tcache_entry_addr >> 12)
```

把 freed tcache entry 的 fd 改为目标地址后，连续两次 `malloc` 使第二次 sheep 指针落到目标；再用负索引升级写入目标 64 位值。

### 泄露地址并改写 ability dispatch

solver 在 heap 上写入 fake chunk header（包括 `0x1001` size），释放后让其进入 unsorted bin，读取其中的 libc 指针并以 `0x21ace0` 推出 libc base。它再把越界读目标改到 `libc.environ`，获得栈地址，并从栈中恢复 PIE base。

最后向全局 `abilities[0..2]` 写入 `system`，令一个 sheep 的 `wps` 为 `/bin/sh\0`。所有 ability 类型都指向 `system` 后，`update_state` 取函数指针并把 `sheep_t *` 传入；该结构首字段正是 `/bin/sh`，等价于执行 `system("/bin/sh")`。

### 验证

题目配置的 flag 是 `DUCTF{y0u_ar3_the_gre4t3st_sheph3rd!!}`。本文未连接服务或执行复杂 heap grooming；索引、泄露减数、safe-linking 公式和函数表终点均由源码与官方 solver 静态对照。

## 方法总结

- 核心技巧：数组上界检查只有单侧时，负索引往往比正向越界更适合抵达相邻控制数据；结合算术升级可完成精确写入。
- 识别信号：`idx >= max`、没有 `idx < 0`、释放后仍可通过越界项读写，以及 function-pointer dispatch，是完整 heap 利用链的组合信号。
- 复用要点：safe-linking 需要 entry 地址参与异或；unsorted-bin 和 libc 偏移均绑定给定 glibc，先分别验证 heap、libc、stack 与 PIE 泄露再做最终函数表覆盖。
