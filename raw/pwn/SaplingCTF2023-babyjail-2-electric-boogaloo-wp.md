# babyjail 2: electric boogaloo

## 题目简述

第二版在进入交互环境前删除了 flag 变量，但秘密字符串曾经被创建并被放入容器对象。删除变量名只减少一个引用，不等于安全擦除；仍被引用或由垃圾回收器跟踪的对象可以通过 gc 模块枚举。

## 解题过程

在控制台导入 gc，遍历 gc.get_objects()，筛选字符串、列表、字典等可能持有 flag 的对象。由于普通不可变字符串未必单独出现在 GC 列表中，更稳妥的是同时检查容器的 repr：

~~~python
import gc

for obj in gc.get_objects():
    try:
        if "maple{" in repr(obj):
            print(repr(obj))
    except Exception:
        pass
~~~

题目构造使秘密仍可从受跟踪的对象关系中找到，最终得到：

~~~text
maple{gc_0r_m3m_dump?}
~~~

## 方法总结

del name 只解除名称绑定，不保证对象立即释放，更不保证内存被覆盖。Python 沙箱若与秘密共享同一解释器，攻击者可能通过 GC、frame、异常回溯或对象图重新取得引用。真正的隔离应把秘密放在不同进程和权限边界中，而不是依赖变量删除。
