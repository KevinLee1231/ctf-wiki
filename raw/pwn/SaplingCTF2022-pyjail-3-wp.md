# PyJail 3

## 题目简述

第三版在 PyJail 2 的基础上再禁止 getattr、globals、update，但核心能力没有变化：open、字符串字面量拼接、文件迭代、strip 和 print 仍然开放。新增禁词没有覆盖上一题的最短读取链。

## 解题过程

逐项对照检查器可见，它仍然只是对 userinput.lower() 做 in 判断。官方上一题 payload 原样通过：

~~~python
print([l.strip() for l in open("f"+"lag.tx"+"t")])
~~~

解析阶段相邻表达式中的加号会把 f、lag.tx、t 合成 flag.txt；黑名单检查时却从未看到连续的 flag 或 txt。文件内容包含许多诱饵文本，筛选 maple{...} 行得到：

~~~text
maple{Did_u_kn0w_that_d0lph1ns_sl33p_with_one_eye_open}
~~~

仓库的 ctfd.json 仍误写成 Pyjail 2 并重复了错误 flag，不能用元数据覆盖实际 hosted/flag.txt。

## 方法总结

在没有消除危险能力的前提下继续追加禁词，只会形成“补洞式”防护。防守方应先定义允许的最小语言，例如只允许数值常量和算术运算，再对 AST 节点、名称和调用目标做正向校验；执行进程仍需独立沙箱，因为语法过滤不应成为唯一安全边界。
