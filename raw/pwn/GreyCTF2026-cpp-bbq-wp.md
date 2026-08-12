# CPP-BBQ

## 题目简述

程序用 `std::shared_ptr<Item>` 管理 Sword、Potion 和 Rock，并把战斗奖励写入容量 $2^{20}$ 的自制环形缓冲区。`RingBuffer::emplace()` 使用 placement new 覆盖槽位，却从不显式析构原对象；槽位回绕时，旧 `shared_ptr` 的引用计数不会减一。通过累计 $2^{32}-1$ 次 Potion 奖励，可以让 GCC/libstdc++ 的 32 位引用计数回绕，制造悬空 `shared_ptr`，再重占堆块并伪造虚表调用 `win()`。

## 解题过程

战斗第 $r$ 轮的正确选项是：

$$
oracle(r)=(r^2+r+1)\bmod3
$$

连续获胜时 `reward_count` 按 1、2、4、… 翻倍，结束战斗后向通知环逐个 `emplace` 同一个 Potion `shared_ptr`。依次进行 32 场战斗，第 $k$ 场恰好赢 $k+1$ 轮并选择结束，可分别加入 $2^k$ 条通知：

$$
\sum_{k=0}^{31}2^k=2^{32}-1
$$

环缓冲区回绕后，新对象直接 placement-new 到仍存活的 `ItemReceivedNotification` 上。构造新 `shared_ptr` 会递增 Potion 控制块的 `_M_use_count`，但被覆盖的旧 `shared_ptr` 没有析构，引用不会抵消。原始计数 1 加上 $2^{32}-1$ 后回到 0。

再开始一场故意输掉的战斗。函数局部变量 `auto reward = items[1]` 先把计数从 0 加到 1，函数返回时析构又从 1 减到 0，于是 `make_shared<Potion>` 的约 `0x50` 堆块 `P` 被释放到 tcache；全局 `items[1]` 仍保存 `P+0x10`，形成 UAF。

接下来进入 blacksmith：

1. `buy 2` 把正常 Rock 放入 `inventory[0]`。
2. `rename 0 <0x40-byte payload>` 让 Rock 的 `std::string` 申请 0x41 字节，对应 0x50 堆块，恰好从 tcache 取回 `P`。
3. 在该字符串内容的 `P+0x10` 位置布置伪 `Item` 和伪 `std::string`。
4. `buy 1` 把悬空 Potion 指针放入 `inventory[1]`。
5. 借伪 string 的数据指针完成一次定址写，再 `inspect 1` 触发虚调用。

伪对象的关键布局为：

```text
P+0x10  fake vptr = AVT
P+0x18  name._M_p = AVT+0x10
P+0x20  name._M_size = 8
P+0x28  name._M_capacity = 0x1000
```

`AVT` 选在固定、可写的通知 BSS 区附近。较大的 capacity 使随后 `rename 1` 不再分配内存，而是把输入原地复制到 `name._M_p`，于是：

```text
rename 1 p64(win)  ->  *(AVT+0x10) = win
inspect 1          ->  call *(*(P+0x10)+0x10) = win
```

目标无 PIE，`win()` 地址固定；脚本还检查所有二进制地址字节均不含空白，因为 `std::cin >> name` 会在空白处停止。成功后 `win()` 执行 `cat flag.txt`：

```text
grey{migrating_to_rust_soon_8bc766f8436fb0db}
```

远端主要耗时来自约 $2^{32}$ 次 `emplace`，官方说明优化后的完整过程约需十分钟，不能把本地缩短引用计数的调试版本当成正式复现。

## 方法总结

placement new 只负责构造，不会自动结束被覆盖对象的生命周期。环形容器若存放带资源所有权的 C++ 对象，覆盖前必须调用析构；否则 `shared_ptr` 计数会被单向放大。计数回绕先制造 UAF，`std::string` 的可控分配再完成同尺寸重占，最后利用伪 string 获得 BSS 定址写并伪造虚表。整个链条不需要地址泄漏，依赖的是固定二进制布局和明确的堆尺寸匹配。
