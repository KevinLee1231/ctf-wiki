# spawn-shell

## 题目简述

程序读取一行字符串后直接调用 system(buf)。题目没有额外过滤、内存破坏或命令拼接障碍，因此输入 shell 命令即可执行。

## 解题过程

连接服务后直接发送：

~~~bash
cat flag.txt
~~~

也可以先发送 /bin/sh 再交互执行 cat flag.txt。服务输出：

~~~text
maple{bin_sh_cat_flag_txt}
~~~

## 方法总结

把不可信输入直接交给 system 等价于完整 shell 命令注入。本题最短路径就是执行读取 flag 的命令，不需要构造 ROP 或 shellcode。修复应避免调用 shell，改用固定程序和参数数组，并配合最小权限运行。
