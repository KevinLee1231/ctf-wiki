# mybookv1

## 题目简述

图书管理程序在创建图书时先把新 `Book` 指针写入全局数组，再为描述分配内存。如果描述的 `malloc` 失败，函数只退出当前分支，却把带有未初始化 `desc` 指针的图书留在数组中。释放块残留数据因此可被当作指针读取。

## 解题过程

先通过正常申请、释放塑造堆布局，再创建图书并把描述长度设为极大值，使 `malloc(desc_len)` 返回 `NULL`。新 `Book` 复用旧堆块，但 `desc` 没有初始化；打印图书时，程序会沿残留指针输出内存。重复这一过程分别取得 heap 和 unsorted-bin/libc 泄露。

有了两个基址后，官方 exploit 伪造 `Book` 与 `0x70` 大小的空闲块，操纵 tcache 的 `fd` 指向 `__free_hook`。随后把 `system` 写入 hook，并释放描述内容为 `/bin/sh` 的图书：

```text
free("/bin/sh") -> __free_hook("/bin/sh") -> system("/bin/sh")
```

取得 shell 后 flag 为：

```text
n00bz{miss_error_handling_turns_into_RCE?}
```

## 方法总结

失败路径必须恢复对象不变量。这里真正的问题不是单纯“malloc 失败”，而是全局数组已经发布了一个未完成初始化的对象。堆利用时应分别证明泄露来源、伪造对象指向和 tcache 尺寸链，而不是把它们统称为 UAF。
