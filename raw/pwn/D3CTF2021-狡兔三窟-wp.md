# 狡兔三窟

## 题目简述

本题是 C++ 智能指针误用导致的 UAF。`NoteStorageImpl::backup` 对 `unique_ptr` 调用 `get()` 后把裸指针传给 `NoteDBImpl`，而 `NoteDBImpl` 构造函数把该裸指针保存为成员；之后 `delHouse` reset 原智能指针，`NoteDBImpl` 中保存的就是悬垂指针。

题目附件包含 C++ 源码、二进制和 exp，核心机制是 `unique_ptr` 管理对象时仍向外暴露了裸指针，形成生命周期失配。解题主线是用 `vector` 的扩容和 `shrink_to_fit` 控制重新申请，回收原 `0x350` 对象块，利用残留 gift 泄露堆和 libc，最后伪造 vtable 调用 one_gadget。

## 解题过程

题目源码/exp 见 [`sadmess/easy_cpp`](https://github.com/sadmess/easy_cpp)。下文保留了该仓库中的关键对象关系：`unique_ptr::get()` 造成裸指针逃逸、`NoteDBImpl` 保存悬垂指针，以及通过 `vector` 占位和 vtable 劫持完成利用。

智能指针误用造成指针悬垂

在 `NoteStorageImpl::backup` 中可以看到如下操作：对智能指针使用 `get()` 后，用这个裸指针新建 `NoteDBImpl` 对象。

```cpp
v2 = std::unique_ptr<NoteImpl, std::default_delete<NoteImpl>>::get(this);
std::make_unique<NoteDBImpl, NoteImpl *>(&v3, &v2);
```

而该对象的构造函数又在类属性中保存了该指针

```cpp
v2 = *(NoteImpl **)std::forward<NoteImpl *>(a2);
v3 = (NoteDBImpl *)operator new(0x10uLL);
NoteDBImpl::NoteDBImpl(v3, v2);

// NoteDBImpl::NoteDBImpl(this, note)
*(_BYTE *)this = 0;
*((_QWORD *)this + 1) = note;
```

此时调用NoteStorageImpl::delHouse,reset智能指针后,NoteDBImpl中保存的便是一个悬垂指针

将悬垂指针所指内存用vector申请回来

简单调试即可发现释放的对象大小为0x350

```text
Tcachebins[idx=51, size=0x350] count=1
  Chunk(addr=0x55555576de90, size=0x350, flags=PREV_INUSE)
```

调用 `NoteStorageImpl::editHouse` 可通过 `vector` 申请内存。GCC 下 `vector` 的申请机制是：当写入超过当前 `capacity` 时，会重新申请当前 size 两倍左右的新空间。
此时直接输入大量字符扩展vector是无法申请到0x350的堆块的。

进一步逆向可发现 `NoteStorageImpl::saveHouse` 提供了 `shrink_to_fit` 功能。通过计算 `0x342 / 8 = 0x68`，第一次申请 `0x68` 后调用 `shrink_to_fit`，即可控制 `vector` 大小为 `0x68`；再加入三次 `0x68` 个字符，最后 add 一个字符即可申请到 `0x350` 的堆块。

泄露地址&获得shell

申请到0x350的堆块后,经过查看释放的对象内存即可发现其中残留着gift信息,即堆地址和libc地址,

```text
0x55555576de80: 0x000055555576edd0  0x0000000000000351
0x55555576dea0: 0x000055555576e200  0x000055555576e200
...
0x55555576e040: 0x000055555576e1e5  0x00007ffff74da0e0
```

调用 `NoteStorageImpl::show` 即可泄露对应信息。需要注意的是，`show` 只能在智能指针 reset 后调用；此时虚表已经清零，直接调用 `printf` 无法打印任何信息，因此需要先把堆块申请回这里并填充非 0 字节，才能泄露处在堆块中部的地址。

最后覆盖 vtable 地址为当前堆块，并在伪造 vtable 中写入 one_gadget，再调用 `NoteStorageImpl::encourage` 触发虚表函数调用即可拿 shell。

exp 的关键是用容器申请模式回收被释放的对象块，使裸指针指向攻击者可控内容；之后通过残留字段泄露地址并伪造 vtable。

## 方法总结

- 核心技巧：智能指针泄露裸指针后生命周期失配，配合 `vector` 可控扩容把释放对象块重新占位，形成 UAF 到 vtable 劫持。
- 识别信号：`unique_ptr::get()` 的结果被长期保存到其他对象，随后原 owner 被 reset；释放块大小可通过调试确认，且程序提供可控大小/内容的容器写入。
- 复用要点：C++ 题里智能指针不等于没有 UAF；要看裸指针是否逃逸出 owner 生命周期。占位时需要理解容器扩容策略，必要时用 `shrink_to_fit` 调整 capacity。

