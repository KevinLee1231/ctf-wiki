# mapmap2

## 题目简述

附件是一个静态链接、未优化且已去符号的 C++ 程序。主函数要求输入恰好 268 个字符，并只接受 `a`、`d`、`s`、`w`。程序把每个字符的高四位、低四位分别作为两层容器的 key：

```text
std::map<int, std::unordered_map<int, Node *>>
```

查表所得的 `Node *` 是下一状态；从全局 `map_entry` 走到 `map_end` 后，程序输出 `md5(input)` 形式的 flag。因此题目本质是把运行时构造的嵌套 STL 容器恢复成有向图，再求入口到出口的最短路径。

## 解题过程

主逻辑可以还原为：

```cpp
std::string input;
std::cin >> input;
Node *pos = map_entry;

if (input.size() != 268)
    fail();

for (unsigned char c : input) {
    if (c != 'a' && c != 'd' && c != 's' && c != 'w')
        fail();

    int hi = c >> 4;
    int lo = c & 0xf;
    pos = (*pos)[hi][lo];
    if (pos == nullptr)
        fail();
}

if (pos == map_end)
    print("SUCTF{" + md5(input) + "}");
```

初始化函数非常大，静态逐项还原性价比很低。更直接的方法是在初始化完成、读取用户输入之前暂停程序，把堆内存导出。官方附件版本使用：

```gdb
b *0x462D8C
r
x/2gx 0x63F530
info proc mappings
```

一次运行中得到：

```text
map_entry = 0x65b3d0
map_end   = 0x667b50
heap      = 0x640000 .. 0x730000
```

随后导出堆：

```gdb
dump memory heap 0x640000 0x730000
```

地址应以当前调试会话实际输出为准。即使程序 No PIE，也不应在没有核对映射的情况下盲用另一台机器记录的堆边界。

该二进制使用的 libstdc++ 红黑树节点布局可按下面的偏移解析：

```text
std::map object:
  +0x08  nil/header
  +0x10  root
  +0x18  minimum
  +0x20  maximum

tree node:
  +0x08  parent
  +0x10  left
  +0x18  right
  +0x20  int key
  +0x28  embedded unordered_map value
```

内层 `unordered_map<int, Node *>` 的链表头位于对象 `+0x10`；节点的 `next`、key、value 分别位于 `+0x00`、`+0x08`、`+0x10`。下面的脚本不依赖 NetworkX，直接解析 dump 并做 BFS：

```python
#!/usr/bin/env python3
from collections import deque
from hashlib import md5

HEAP_BASE = 0x640000
MAP_ENTRY = 0x65B3D0
MAP_END = 0x667B50

heap = open("heap", "rb").read()

def read(addr, size):
    off = addr - HEAP_BASE
    assert 0 <= off <= len(heap) - size
    return heap[off:off + size]

def u32(addr):
    return int.from_bytes(read(addr, 4), "little")

def u64(addr):
    return int.from_bytes(read(addr, 8), "little")

def parent(node):
    return u64(node + 0x08)

def left(node):
    return u64(node + 0x10)

def right(node):
    return u64(node + 0x18)

def successor(map_addr, node):
    nil = map_addr + 0x08
    r = right(node)
    if r not in (0, nil):
        cur = r
        while left(cur) not in (0, nil):
            cur = left(cur)
        return cur

    cur = parent(node)
    while right(cur) == node:
        node, cur = cur, parent(cur)
    return cur

def map_nodes(map_addr):
    nil = map_addr + 0x08
    root = u64(map_addr + 0x10)
    if root in (0, nil):
        return

    cur = u64(map_addr + 0x18)
    maximum = u64(map_addr + 0x20)
    while True:
        yield cur
        if cur == maximum:
            break
        cur = successor(map_addr, cur)

def unordered_nodes(obj):
    cur = u64(obj + 0x10)
    while cur:
        yield cur
        cur = u64(cur)

def edges(state):
    for outer in map_nodes(state):
        hi = u32(outer + 0x20)
        inner_map = outer + 0x28
        for inner in unordered_nodes(inner_map):
            lo = u32(inner + 0x08)
            nxt = u64(inner + 0x10)
            if nxt:
                yield nxt, (hi << 4) | lo

queue = deque([MAP_ENTRY])
previous = {MAP_ENTRY: None}

while queue:
    state = queue.popleft()
    if state == MAP_END:
        break
    for nxt, byte in edges(state):
        if nxt not in previous:
            previous[nxt] = (state, byte)
            queue.append(nxt)

assert MAP_END in previous
path = bytearray()
cur = MAP_END
while previous[cur] is not None:
    cur, byte = previous[cur]
    path.append(byte)
path.reverse()

assert len(path) == 268
assert set(path) <= set(b"adsw")
print(path.decode())
print(f"SUCTF{{{md5(path).hexdigest()}}}")
```

求得的路径为：

```text
ddssaassddddssddssssaassaawwwwwwaassssssssssddssssssddwwddddssssddssssddwwwwddssddwwddssddwwwwwwwwwwaawwaawwddwwaaaawwwwddwwddssddddddssaassaassddssddwwddwwwwwwwwwwwwddssssssssssddddssssssddwwwwwwddssddssaassssddssssaaaawwwwaassssaawwaassssssddddssssddssssssaassdddddd
```

其长度为 268，MD5 为：

```text
8b587367b99e5e2fcbdb6598da14b9bc
```

最终 flag：

```text
SUCTF{8b587367b99e5e2fcbdb6598da14b9bc}
```

## 方法总结

本题的难点不在迷宫搜索，而在运行时 STL 对象恢复。巨大的初始化函数只是把图编码成大量容器操作；在初始化完成后导出堆，可以绕开无价值的逐条反编译。

解析时必须使用与目标 libstdc++ ABI 对应的对象偏移，并区分 map 的 header、树节点和内层 unordered_map 节点。原始官方脚本只读取低 32 位指针，在该 No PIE、低地址样本中可以工作，但语义上 `Node *` 是 64 位；上面的版本使用 `u64`，避免把偶然的低地址布局误写成通用结构。最后还要用长度、字符集合和程序计算出的 MD5 三重校验路径。
