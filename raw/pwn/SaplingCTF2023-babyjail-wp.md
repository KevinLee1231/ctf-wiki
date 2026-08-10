# babyjail

## 题目简述

服务先把 flag 读入全局变量，再调用 code.interact() 提供交互式 Python 控制台。它没有真正删除对象或隔离解释器状态，因此只需从控制台读取全局命名空间、调用栈或当前脚本内容。

## 解题过程

最直接的方法是在交互控制台查看 globals()；若启动方式改变了可见命名空间，也可以沿 frame 回到调用 code.interact() 的外层：

~~~python
import inspect
inspect.currentframe().f_back.f_locals
~~~

另一条稳定路线是读取正在运行的脚本：

~~~python
open(__import__("sys").argv[0]).read()
~~~

源码或外层局部变量中可见：

~~~text
maple{50ln_l0n63r_7h4n_5rc?}
~~~

## 方法总结

Python 交互控制台不是沙箱。只要 import、文件读取、frame 或原有 globals 仍可访问，秘密就没有隔离。分析 pyjail 时应先列出攻击者已经拥有的对象和命名空间，再考虑复杂的类层次逃逸；本题三行代码本身就暴露了全部上下文。
