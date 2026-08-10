# Echowo

## 题目简述

程序把 flag 读到堆上，并把其指针留在 printf 调用可访问的参数位置。用户名被直接用作格式字符串，因此无需写内存或构造 ROP，只要用位置参数 %s 解引用该指针即可。

## 解题过程

先发送多个 %p 枚举栈参数，观察第 7 项对应一段可读堆地址。把格式符改为位置字符串读取 <code>%7$s</code>：

~~~python
io.sendlineafter(b"What's your name?\n", b"%7$s")
~~~

<code>%7$s</code> 将第 7 个参数当作 char *，持续打印到 NUL 终止。该指针正好指向预先读取的 flag，输出：

~~~text
maple{fowmat_stwing_vuwnewabiwity!!}
~~~

Full RELRO、栈 Canary、NX 和 PIE 都不妨碍信息泄漏，因为利用没有修改控制流。

## 方法总结

格式字符串侦察通常从 %p、%lx 开始定位参数，再用 %s 解引用可疑指针。%s 可能因无效地址导致崩溃，应逐项测试或限制长度，例如 %.64s。最根本的修复仍是固定格式串，并避免让敏感指针在不必要的可变参数调用现场存活。
