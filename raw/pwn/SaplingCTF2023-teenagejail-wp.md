# teenagejail

## 题目简述

服务清空或替换 sys.modules，并限制用户只能提交一行 Python 表达式。flag 仍存在于外层调用帧的局部变量中；目标是在不依赖正常 import 的情况下取得一个 frame 对象并向上遍历。

## 解题过程

生成器对象会暴露 gi_frame。利用列表在构造生成器后形成自引用，再取生成器当前帧的 f_back，可以跨回执行用户代码的外层。官方思路可压缩为一行：

~~~python
l=[];l+=[(g.gi_frame.f_back.f_back for g in l)];print(next(l[0]).f_locals["flag"])
~~~

这里 l 在生成器真正执行前已包含生成器本身，因此表达式中的 g 就是该生成器。gi_frame 取得生成器帧，连续 f_back 回到沙箱包装层和持有 flag 的调用层，再从 f_locals 读取：

~~~text
maple{41n'7_py7h0n_r34l_fun?}
~~~

也可以从可达类的方法 __globals__ 找回真实 sys，再检查 frame；但生成器帧路线更直接。

## 方法总结

移除 sys.modules 并不能消除解释器中已经存在的 frame、函数全局变量和类型对象。评估 Python 沙箱时必须把对象能力而非模块名称作为边界：只要能构造或找到 frame，就可能访问上层局部变量；只要能找到带 __globals__ 的 Python 函数，就可能恢复被删除的模块引用。
