# mstr

## 题目简述

题目用 Python 3.12.4 和 `ctypes` 实现了一个“可变字符串”容器。程序把 `str` 对象的地址记录为 `base_ptr`，并按固定偏移直接改写对象内容与长度：`base_ptr + 0x10` 被当作长度字段，`base_ptr + 0x28` 被当作字符访问基址。用户可以创建、拼接、打印和逐字节修改字符串，还能用另一个字符串表示拼接上限。

漏洞不在常规 Python 语义，而在“CPython 假定 `str` 不可变”和题目“用 `ctypes` 原地改写 `str`”之间的冲突。CPython 会复用部分单字符字符串对象；当同一个驻留对象既是某个 `MutableStr.data`，又是另一个对象的 `max_size_str` 时，原地拼接前者会同步篡改后者看到的上限。由此可突破边界检查，破坏相邻 Python 对象，并继续构造地址泄露与代码执行。

## 解题过程

### 1. 审计边界检查

`set_max_size` 先把输入转成整数检查，却把原始字符串对象直接保存下来：

```python
def set_max_size(self, max_size_str):
    if int(max_size_str) < ((len(self) + 7) & ~7):
        self.max_size_str = max_size_str
```

真正拼接时又重新执行 `int(self.max_size_str)`。只要检查后能改变该字符串对象的内容，先前的限制就失效：

```python
if self.max_size_str == "":
    max_size = (len(self) + 7) & ~7
else:
    max_size = int(self.max_size_str)
```

可用如下序列触发：

```text
new BBBBBBBBBBBBBBBB
new AAAAAAAAAAAAAAAA
new CCCCCCCCCCCCCCCC
new 5
set_max 5 1
+= 3 3
+= 3 3
```

`new 5` 得到的单字符字符串与 `set_max` 保存的字符串是同一个 CPython 运行时对象。对索引 3 执行自拼接，会把共享对象从 `"5"` 原地扩成 `"55"`、再扩成更长的十进制数；索引 1 随后读取到的 `max_size` 因而远大于初次检查时的 5。这里不能把漏洞简化为“整数校验错误”：决定性条件是单字符对象复用与非法原地修改同时存在。

### 2. 从越界拼接得到相邻对象破坏

`_add_str` 通过 `ctypes` 逐字节复制，却只依据已被放大的 `max_size` 判断范围，并在完成后直接增加目标 `str` 的长度字段。利用同尺寸字符串整理 CPython 小对象分配布局，使 `A` 与待攻击的 `C` 落在可预测的相对位置，再把构造好的数据 `+=` 到 `A`，即可覆盖 `C` 的对象头。

题目输入经 `input().split()` 解码，直接创建字符串不便携带任意高位字节。单题 exploit 的做法是先创建等长占位字符串，再用 `modify` 逐字节改成目标载荷，最后通过越界拼接复制过去。由于地址高字节、UTF-8 替换字符和小对象分配位置会随运行变化，脚本还需要枚举一个未知高字节并在失败时重连。

### 3. 单题 WP 的类型伪造路线

题目目录中的 exploit 把越界能力依次转换为三次泄露和一次函数指针劫持：

1. 改坏相邻 `str` 的头部与长度，打印该对象，从输出中的 `0x...` 地址取得被攻击对象自身地址。
2. 在已知堆地址附近布置伪造 `PyTypeObject`，让对象的 `ob_type` 指向它，并令 `tp_name` 指向 Python 映像内的已知位置。再次打印对象，利用类型名格式化过程泄露 PIE 指针，据固定版本偏移还原 Python 可执行文件基址。
3. 由 PIE 基址定位 `PyByteArray_Type` 和 `dup2@GOT`。把同一片可控内存重解释为伪造 `PyByteArrayObject`，令 `ob_bytes`、`ob_start` 指向 `dup2@GOT`；打印 `bytearray` 即可读出真实 `dup2` 地址，再按随附件提供的 libc 偏移计算 libc 基址和 `system`。
4. 在伪造类型中把 `tp_repr`、`tp_str` 改为 `system`，同时让待打印对象起始处包含 shell 命令。执行 `print` 时，CPython 经类型槽调用 `system(object_address)`，获得命令执行，随后读取 flag。

这条路线依赖附件中固定的 Python 3.12.4 与 libc，`PyTypeObject` 字段偏移、`PyByteArray_Type`、GOT 和函数偏移都不能脱离对应二进制照搬。

### 4. 总 PDF 中的 FSOP 替代路线

总 PDF 给出的 Nu1L 解法使用同一根因，但没有伪造 Python 类型。它先批量申请相同大小的字符串，通过越界字符串改写高地址字符串的长度，形成覆盖大范围地址空间的重叠对象：

1. 从扩大后的打印结果中泄露 `PyUnicode_Type` 等指针，定位 Python 映像及相邻映射。
2. 把一个重叠字符串的长度改成极大值，使 `print` 充当越界读、`modify` 充当逐字节越界写，进而定位 libc。
3. 在可写内存布置伪造 `_IO_FILE`、wide-data、`setcontext` 上下文和后续 ROP/ORW 载荷。
4. 把 libc 的 `_IO_list_all` 改到伪造对象；关闭发送方向触发标准流清理，进入 `setcontext`，最后打开并输出 flag 文件。

PDF 脚本需要猜测覆盖范围和对象间距，本地标注成功率约为一半，远端更低，并通过每次失败后调整候选长度重连。这个不稳定性来自 CPython 分配布局和运行环境差异，不是漏洞原语本身消失；实战中应先用附件容器标定对象距离，再把偏移枚举限制在最小范围。

## 方法总结

- 根因是题目用 `ctypes` 破坏了 CPython 对 `str` 不可变性的基本假设，而 `max_size_str` 又保留共享字符串对象，不是单纯的 Python 逻辑越界。
- 利用链的第一道门槛是借单字符对象复用把小上限变成大整数；第二道门槛是稳定整理小对象布局并覆盖相邻对象头。
- 得到对象破坏后有两条已验证路线：伪造 `PyTypeObject` 与 `PyByteArrayObject` 完成 PIE/libc 泄露并劫持 `tp_repr`，或扩大重叠字符串形成任意读写后打 FSOP。
- Python 内部结构、GOT、libc 和类型槽偏移都与附件版本绑定；远程不稳定时应重连并枚举布局相关的少量未知量，而不能把某次调试地址硬编码为通用值。
