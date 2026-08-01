# GlacierCTF 2025 hyprice

## 题目简述

题目用自制的 Hyprutils 风格智能指针和动画变量实现一个 C++ rice/dotfile 管理器。`CRice::fork()` 让两个 Rice 共享同一个 `m_score`，而分数动画结束回调以 `[this]` 捕获原对象。删除原 Rice 后，共享的动画对象仍然存活，回调最终访问已释放的 `this`，形成 UAF。

由于 `CRice` 与 `CDotfile` 分配大小相同，可让 Dotfile 占据释放的 Rice chunk。回调对 `m_name` 的写入会覆盖 Dotfile 起始处 `std::string` 的指针，由此构造任意读写并劫持函数指针。

## 解题过程

### 1. 制造延迟触发的 UAF

`fork` 的关键逻辑不是深拷贝分数对象，而是：

```cpp
auto res = makeUnique<CRice>(...);
res->m_score = m_score;
```

当 unsigned score 通过多次减分发生下溢，程序为动画注册结束回调：

```cpp
m_score->setCallbackOnEnd([this](auto) {
    m_name.put(g_hyprice->m_trophyChar.view());
});
```

创建 Rice A，fork 出 C 后让两者共享 score；触发下溢和动画，再删除 A。C 持有的共享引用保证动画继续存在，但 lambda 内的 `this` 仍指向已释放的 A。等动画结束，回调便对旧 A 地址执行写入。

### 2. 用同尺寸对象把 UAF 变成任意读写

紧接着创建 CDotfile B，使 allocator 将其放入 A 的旧 chunk。`CDotfile` 的第一个成员是 `std::string`，布局开头包含字符串数据指针；UAF 回调现在把全局可控的 trophy 字符串写到这个位置。

配合程序接受 `std::string_view` 的接口，可以只改指针的低位而不破坏其余元数据。先把 B 的字符串指针引到同一堆区附近，再使用 `yank` 的 Base64 输出读取指针目标，得到堆地址。随后逐步把指针改到：

- 堆中对象或 vtable/函数指针，恢复 PIE 基址；
- 程序可达的 `stdout` 等 libc 指针，恢复 libc 基址；
- 任意目标地址，配合编辑功能写入攻击数据。

这条利用不依赖 tcache freelist 劫持；allocator 只负责让两个同尺寸 C++ 对象重叠，真正的读写原语来自 `std::string` 指针被 UAF 回调改写。

### 3. 覆盖智能指针 deleter

参考解选择 `CUniquePointer::Impl::deleter_` 作为控制流目标，它的调用签名是 `void(char *)`，与 libc 的 `system(char *)` 足够匹配。先用任意写把该函数指针改为 `system`，再借 trophy 写入让另一个 Rice 对象的开头保存：

```text
/bin/sh\0
```

删除该对象时，自制 unique pointer 调用被篡改的 deleter，于是执行 `system("/bin/sh")`。进入 shell 后读取 `/flag.txt`，源码实例为：

```text
gctf{CppChadSolvesHyprPwnChallengeAndLearnsAboutOSC52}
```

## 方法总结

共享所有权只能延长被共享对象的寿命，不能自动延长 lambda 捕获的裸 `this`。这里 score 仍存活，恰恰让针对已释放 Rice 的回调延迟到可控重分配之后发生。利用时先证明对象大小相同和回调触发顺序，再把 C++ 对象布局转化为 `std::string` 指针读写，最后选择签名兼容的 deleter 完成控制流劫持。修复应让动画变量由唯一所有者管理，回调使用可验证生命周期的弱引用，并避免无意义的 unsigned 分数下溢。
