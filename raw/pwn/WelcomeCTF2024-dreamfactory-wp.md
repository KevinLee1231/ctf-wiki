# dreamfactory

## 题目简述

程序有两套共用堆分配器的功能：Dream 分配函数指针数组并逐项调用，Note 分配可读写缓冲区。释放后的堆块没有清零。目标是利用同尺寸堆块复用，先泄露 PIE 函数地址，再把隐藏的 `dream_about_flag_real` 地址残留到新的函数指针数组中并触发调用。

## 解题过程

### 泄露函数地址

先创建若干 Dream，使程序在堆上分配函数指针数组并填入公开的梦境函数地址；执行 Dream 后数组被释放。随后创建相同尺寸的 Note，分配器会复用刚释放的块。

glibc tcache 只会覆盖空闲块开头的元数据，后面的旧函数指针仍在。Note 内容只写到这些残留地址之前，再读取 Note，`printf("%s")` 会继续输出旧数据，从而泄露 `dream_about_valorant` 等函数的运行时地址。利用符号间固定偏移计算隐藏函数：

```python
win = leaked_valorant \
    + elf.sym.dream_about_flag_real \
    - elf.sym.dream_about_valorant
```

### 伪造函数指针

再次分配 Note，在 tcache 元数据后的目标槽位写入 `win`，然后释放该 Note。接着申请相同尺寸的 Dream 数组，只填充前面的合法槽位，让先前写入的 `win` 留在后续未初始化槽位：

```python
note = b"A" * 24 + p64(win)
```

`start_dreaming` 遍历数组时会依次调用这些指针，最终执行 `dream_about_flag_real`，输出：

```text
grey{i_dreamt_about_the_flag_appearing_in_my_dreams}
```

## 方法总结

释放内存只代表分配器可再次使用它，并不保证数据被抹除。本题把同一旧数据先用作信息泄露、再用作函数指针注入。审计堆对象时应关注跨类型同尺寸复用、未初始化内容和间接调用三者的组合。
