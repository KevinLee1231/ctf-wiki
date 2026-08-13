# fake-blockchain

## 题目简述

程序把输入的每个字符包装成一个单向链表块，新块插入链表头。每个块保存字符、当前哈希和前驱指针；查看链表时，程序从最后输入的字符向前遍历，并要求哈希序列与全局 `EXPECTED` 数组一致。

## 解题过程

正向哈希函数为：

```c
unsigned int calculate_hash(char c, unsigned int prev_hash) {
    return (prev_hash + c) ^ 0x1234;
}
```

设当前期望哈希为 $h_i$、前一块哈希为 $h_{i-1}$，则字符可直接反演为 $c_i = (h_i \mathbin{\mathrm{xor}} 0x1234) - h_{i-1}$。由于链表头保存最后输入字符，而 `EXPECTED` 按链表遍历顺序排列，恢复时要先反转数组：

```python
expected = [
    6383, 2654, 6129, 1428, 5945, 1176, 5709, 1044, 5550, 1894,
    5363, 1682, 5170, 1443, 5922, 1188, 5660, 949, 4358, 697,
    4136, 426,
]

previous = 0x1337
flag = []
for current in reversed(expected):
    flag.append(chr((current ^ 0x1234) - previous))
    previous = current

print("".join(flag))
```

输出为 `grey{struct5_4re_ug1y}`。向原始程序提交长度 22 和该字符串后，`View blocks` 显示 22 个逆序节点，状态为 `VALID`，所有哈希均与 `EXPECTED` 对齐。

## 方法总结

链表头插入会反转输入顺序，忽略这一点会得到倒置或完全错误的状态递推。先画清“输入顺序、哈希依赖顺序、验证遍历顺序”三者关系，再对可逆的加法与异或逐步反演即可。最终 flag 为 `grey{struct5_4re_ug1y}`。
