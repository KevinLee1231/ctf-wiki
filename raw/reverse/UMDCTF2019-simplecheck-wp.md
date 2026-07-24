# UMDCTF 2019 - SimpleCheck

## 题目简述

服务要求提交三组不同的 key，每组由 13 个无符号字节组成。程序会排序每组数字，再依据内嵌关系表验证；三组都通过后执行 `/bin/sh`。

## 解题过程

逆向校验函数可知，`.rodata` 文件偏移 `0x1300` 保存 39 个 64 位位图。把数字看作顶点 $0$ 到 $38$，第 $u$ 行的第 $v$ 位为 1，表示选择 $u$ 后仍允许选择 $v$。一组合法 key 就是这个有向关系中的 13 顶点团。

可以按升序回溯，并在每次选点后把候选集合与当前顶点的邻接位图相交：

```python
import struct

data = open("challenge", "rb").read()
rows = struct.unpack_from("<39Q", data, 0x1300)

def search(prefix, candidates):
    if len(prefix) == 13:
        yield prefix
        return

    need = 13 - len(prefix)
    for position, vertex in enumerate(candidates):
        if len(candidates) - position < need:
            break
        remaining = tuple(
            other for other in candidates[position + 1:]
            if (rows[vertex] >> other) & 1
        )
        yield from search(prefix + (vertex,), remaining)

for index, key in enumerate(search((), tuple(range(39)))):
    print(*key)
    if index == 2:
        break
```

前三组解为：

```text
0 3 6 11 12 15 18 22 24 27 30 34 37
0 3 6 11 12 15 18 22 24 28 30 34 37
0 3 6 11 12 15 18 22 24 29 30 34 37
```

将三行依次输入原始二进制，程序显示 `SimpleKey validation success` 并启动 shell，证明解法完整。仓库没有服务端 `flag` 文件，原远程服务也已下线，因此无法从现有证据可靠恢复最终 flag；这里不编造具体字符串。

## 方法总结

大量成对合法性检查常可抽象为图问题。本题把 key 转化为固定大小的 clique，升序搜索既去除了排列重复，也便于用剩余候选数剪枝。对于已能在原二进制上验证、但缺少服务端秘密的数据，应清楚区分“利用链已完成”和“历史 flag 不可恢复”。
