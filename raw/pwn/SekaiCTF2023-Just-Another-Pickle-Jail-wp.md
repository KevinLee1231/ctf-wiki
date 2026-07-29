# Just Another Pickle Jail

## 题目简述

服务读取一行十六进制，将其转成字节后交给修改版 Python `_Unpickler`：

```python
up = Unpickler(BytesIO(bytes.fromhex(input(">>> "))))
up.load()
```

常见的 Pickle RCE 路线几乎都被封死：

- `REDUCE`、`NEWOBJ`、`NEWOBJ_EX`、`INST`、扩展 opcode 会进入 `die()`；
- `find_class` 只从 `__main__` 取对象，并过滤 `exec`、`os`、`eval`、`open`、`load` 等子串；
- `BUILD`、`SETITEM(S)` 会过滤包含 `__` 的普通字典键；
- 全局 `getattr`、`setattr`、`__import__` 被替换，许多内建名称又从命名空间删除。

但解释器仍允许 `GLOBAL`、`STACK_GLOBAL`、`BUILD`、`NEXT_BUFFER`、memo 和字典写入。官方 payload 不调用被禁的 `REDUCE`，而是逐步篡改 Unpickler 自身及内建函数，最终让 `STACK_GLOBAL` 的类型检查主动执行代码。

## 解题过程

### 找到 `BUILD` 的 slotstate 缺口

自定义 `load_build` 对普通 state 字典过滤双下划线键，却保留了 tuple 形式的 slotstate：

```python
if isinstance(state, tuple) and len(state) == 2:
    state, slotstate = state

if slotstate:
    for key, value in slotstate.items():
        setattr(inst, key, value)
```

外层自定义 `setattr` 只禁止 `Unpickler.__dict__` 中已经存在的属性名。`__getattribute__` 是从 `object` 继承的，不在创建 `banned` 列表时的类字典中，因此可以通过 slotstate 改写。

payload 把 `Unpickler.__getattribute__` 替换为 `__main__.__getattribute__`，使实例内部访问 `self.find_class`、`self._buffers` 等名称时可以被引向攻击者已经污染的 `__main__` 命名空间。

### 恢复并污染全局命名空间

第一阶段用 `GLOBAL` 取到 `__main__` 模块和 `__builtins__` 字典，再通过 `BUILD` 把被删除的内建名称重新并入模块字典。随后用 `SETITEMS` 写入几个关键替代：

```text
__main__.find_class      = getattr
__main__.persistent_load = print
__main__.str             = True
builtins.str             = True
builtins.next            = BytesIO
```

这一步利用的是 Pickle 虚拟机中的对象引用，不需要在被过滤的 `GLOBAL` 名称里直接出现 `exec` 或 `os`。

接着对服务已经创建的全局 `up` 对象使用 `BUILD`，修改其 `_buffers`，并借助 `NEXT_BUFFER` 把攻击者控制的辅助字节流压入 Pickle 栈。payload 还恢复 `_file_read`、`_file_readline`，清空 `stack`/`metastack`，并换入预先构造的 memo 字典，以便第二阶段只用一串 memo 索引完成对象拼接。

### 劫持 `STACK_GLOBAL` 的类型检查

`load_stack_global` 原本有如下校验：

```python
name = self.stack.pop()
module = self.stack.pop()

if type(name) is not str or type(module) is not str:
    raise UnpicklingError("STACK_GLOBAL requires str")

self.append(self.find_class(module, name))
```

官方生成器先令：

```text
builtins.type = bool
builtins.str  = True
```

对任意非空对象，`bool(value)` 返回单例 `True`，恰好与被替换后的 `str` 是同一个对象，所以 `is not` 判断为假。这使 `STACK_GLOBAL` 可以接收并非字符串的 module/name，从而取得字典的绑定 `__getitem__` 方法。

随后把：

```text
builtins.next = builtins.__dict__.__getitem__
```

并让被劫持的 `self._buffers` 解析为字符串 `"exec"`。于是 `NEXT_BUFFER` 内部本应执行的：

```python
next(self._buffers)
```

实际变成：

```python
builtins.__dict__.__getitem__("exec")
```

返回真正的 `exec` 函数。再用 `SETITEM` 完成：

```text
builtins.type = exec
```

最后一次执行 `STACK_GLOBAL` 时，类型检查中的 `type(name)` 已不再查询类型，而会调用：

```python
exec(name)
```

传入的 `name` 是：

```python
os = object.mgk.nested.__import__("os")
os.system("sh")
```

`chall.py` 早先把原始 `__import__` 藏在 `object.mgk.nested`，本意是供保护代码内部使用，反而为最终 payload 提供了未受限导入。即使后续 Pickle 状态抛出异常，服务的 `try/except` 也会吞掉异常，而 shell 已经生成。

### 生成并提交 payload

官方 `gen-pkl.py` 直接用 `pickle` 模块导出的 opcode 常量拼接上述两阶段程序，最终打印十六进制：

```python
from pickle import BUILD, GLOBAL, NEXT_BUFFER, PROTO, SETITEMS

payload = PROTO + b"\x04"

# 依次拼接：
# 1. BUILD 合并 builtins
# 2. SETITEMS 污染 main/builtins
# 3. BUILD 改写 Unpickler.__getattribute__
# 4. 修改 up._buffers、read/readline、memo 和栈
# 5. 第二阶段 memo 程序劫持 type/str/next
# 6. 最终 STACK_GLOBAL 触发 exec

print(payload.hex())
```

把生成的十六进制发送给服务，在 shell 中执行仅有执行权限的 `./flag`，得到：

```text
SEKAI{Pls_dm_me_your_solution!!!!!_PLS_PLS_PLS_PLS_PLS_PLS_10a429da9dc989a9a30c9d129b8e66abd63749854f80fb23379e5598cba1daaa}
```

## 方法总结

- 核心技巧：不寻找被禁用的 `REDUCE`，而是把仍可用的 `BUILD`、字典写入、memo 与 `NEXT_BUFFER` 组合成解释器状态篡改，最后利用校验代码中的动态内建查找执行 payload。
- 识别信号：保护逻辑修改了全局 builtins，却继续在运行时查找 `type`、`str`、`next`；`BUILD` 的普通 state 与 slotstate 过滤又不一致，这些都是可组合的元对象漏洞。
- 复用要点：安全反序列化不能依赖禁用少数危险 opcode 或字符串黑名单。只要攻击者还能取得可变对象并修改解释器、类、模块或 builtins，原本无害的校验和协议 5 buffer 机制都可能变成调用原语。
