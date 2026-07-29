# Learning OOP

## 题目简述

题目是一个 C++ 宠物管理程序。所有动物对象都含有虚表指针、`char name[0x100]`、年龄、饱食度和状态字段。设置名字时直接执行：

```cpp
std::cin >> this->name;
```

对裸 `char *` 的流提取没有长度限制，因此可以越过 0x100 字节名字缓冲区，破坏相邻堆块。程序又会打印新宠物地址，并在每次菜单操作后自动更新、死亡和 `delete` 对象，提供了堆地址泄漏与可控释放。

## 解题过程

### 1. 获取堆基址

创建第一只 Horse 后，程序直接打印对象指针。官方脚本按固定布局计算：

```python
heap_base = leaked_pet - 0x136d0
```

该偏移取决于题目二进制和启动阶段的分配，不能迁移到其他 libc/构建版本。

### 2. 越界改大相邻块并制造重叠

对象大小约为 0x120。通过给前一只宠物输入超长名字，可以覆盖后一块的元数据。官方利用把相邻块大小改为 `0x481`，再配合创建宠物和反复触发 `update()`，让该大块被释放并与仍在使用的宠物对象范围重叠。

后续创建一只名字填满 0x100 字节的 Horse。`die()` 使用：

```cpp
std::cout << this->name << " died :(";
```

名字没有 NUL 终止时，输出会继续读取重叠块中的 unsorted-bin 指针。官方脚本取泄漏值减去：

```text
0x203b20
```

得到配套 libc 基址。

### 3. 布置伪对象与伪虚表

已知堆和 libc 后，在可控 Horse 名字中放入：

```python
p64(system) + p64(address_of_bin_sh)
```

并选择两个 libc gadget：

```text
gadget1: mov rdi, [rdi+0x10]; call [rax+0x380]
gadget2: call [rax+0x8]
```

官方布局让被破坏对象的虚表槽指向 `gadget1`，伪虚表的后续入口再落到 `gadget2`。同时在对象内构造指向前述 `system` 与 `/bin/sh` 数据的指针。

### 4. 借自动更新触发虚调用

每次菜单操作结束都会遍历宠物：

```cpp
if (pet->fullness_down() == 0 ||
    pet->age_up() > pet->get_max_age()) {
    ...
}
```

`get_max_age()` 是虚函数。覆盖目标对象的 vptr 后，下一次 `update()` 会自动发起虚调用。两级 gadget 调整 `rdi` 并间接调用伪对象中的 `system`，最终执行：

```text
system("/bin/sh")
```

在取得 shell 后读取仓库正式挑战镜像中的 flag：

```text
SEKAI{w0w!!!!!!!!_UM4Z1NG_3xpl0it_sk1llz!!!!}
```

### 5. 版本依赖

利用脚本中的堆偏移、unsorted-bin 偏移和两个 gadget 都绑定官方二进制及 libc。复现时应使用仓库附带的动态链接器和 `libc.so.6`；若改用系统 libc，泄漏计算和 gadget 地址都需要重新求解。

## 方法总结

`std::cin >> char_buffer` 并不会自动知道数组长度，除非显式设置宽度，否则与无界字符串输入一样危险。这里一个简单堆溢出通过对象生命周期扩展成：

```text
堆地址泄漏
→ 相邻块大小破坏
→ unsorted-bin libc 泄漏
→ 重叠对象
→ vptr 劫持
→ system("/bin/sh")
```

C++ 对象的 vptr 位于对象开头，堆重叠后比普通函数指针更容易形成稳定控制流。安全实现应改用 `std::string`，或在读取前用 `std::setw(sizeof(name))` 严格限制长度。
