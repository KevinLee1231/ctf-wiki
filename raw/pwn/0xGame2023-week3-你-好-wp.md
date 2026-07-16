# 你 好

## 题目简述

这是一个 Python jail。程序过滤部分危险字符串，然后把 `sys.stdout` 替换成无论写什么都只输出 `ni hao` 的包装对象，最后执行用户输入。虽然正常标准输出被污染，异常回溯仍写入 `stderr`；Docker 启动脚本又用 `2>&1` 把 `stderr` 合并回网络连接，因此可主动抛出包含全局变量 `flag` 的异常。

## 解题过程

服务端先在重定向输出之前读取 flag：

```python
flag = open("flag.txt").read()
yourinput = input(">>> ")
check(yourinput)

original_stdout = sys.stdout
sys.stdout = helloOutput(original_stdout)
del sys
exec(yourinput)
```

过滤器禁止 `import`、下划线、点号、引号、`eval`、`exec`、`breakpoint` 等字符串，但没有禁止 `raise`、`Exception`、括号或变量名 `flag`。提交：

```python
raise Exception(flag)
```

`exec()` 先求值 `flag`，再抛出未捕获异常。`helloOutput` 只接管了 `sys.stdout`，Python 回溯默认发往 `sys.stderr`；容器中的实际启动命令是：

```bash
python3 -u server.py 2>&1
```

所以异常消息最终仍回到客户端，其中包含 flag。

原稿给出的 `breakpoint()` 只适用于更早的未修补版本。当前仓库的 `check()` 已把 `breakpoint` 明确加入黑名单，不能再作为有效解法。

## 方法总结

语言沙箱不能只盯着命令执行，还要枚举所有输出通道和异常阶段。本题无需绕过导入限制或拿 shell，只需利用 `stdout` 与 `stderr` 的差异完成数据泄露。修复时应避免把敏感对象放进 `exec` 的全局作用域，并统一捕获异常、返回固定错误信息。
