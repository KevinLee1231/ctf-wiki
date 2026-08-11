# DownUnderCTF 2023 SimpleFTPServer Writeup

## 题目简述

服务把 FTP 命令名直接交给 `operator.attrgetter(cmd)(self)`，再把命令后的全部文本作为一个字符串参数调用。由于没有把可调用属性限制为 FTP 命令，攻击者可以沿 Python 对象属性链访问方法的 `__globals__`，而 flag 已在程序启动时读入全局变量 `FLAG`。

## 解题过程

正常命令 `USER anonymous` 等价于：

```python
FTPServerThread.USER("anonymous")
```

`operator.attrgetter` 支持点号分隔的嵌套属性。任意绑定方法都有 `__globals__`，它指向定义该函数的模块全局字典；字典的 `get` 或 `__getitem__` 又恰好接受一个字符串参数，符合服务器固定的调用方式。

连接后直接发送：

```text
__init__.__globals__.get FLAG
```

解析结果等价于：

```python
self.__init__.__globals__.get("FLAG")
```

函数返回值被服务器原样写入控制连接，因此立即得到：

```text
DUCTF{15_this_4_j41lbr34k?}
```

也可以使用 `__init__.__globals__.__getitem__ FLAG`。登录失败、伪装的 vsFTPd banner 和不能正常传输 `flag.txt` 的 `RETR` 都是干扰项，命令分发器在认证前就可访问。

## 方法总结

动态属性分发应使用显式命令表，而不是把用户输入直接交给反射 API。Python 方法对象暴露 `__globals__`，而点号遍历把一次方法选择扩展成任意对象图访问；即使调用参数被固定为单个字符串，仍可选择签名匹配的字典方法泄漏全局状态。
