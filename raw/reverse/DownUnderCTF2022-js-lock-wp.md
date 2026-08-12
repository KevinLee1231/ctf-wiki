# DownUnderCTF 2022 js-lock Writeup

## 题目简述

题目是一页只有 `0`、`1` 和提交按钮的 JavaScript“锁”。页面要求依次解锁数字 1 到 1337，随后才会用输入派生密钥解密 flag。锁的主体是一个很深的嵌套数组 `LOCK`，因此关键是理解按键如何在数组树中移动，并自动求出每个目标数字的路径。

## 解题过程

阅读按钮事件可以还原状态机：按 `1` 只把当前子元素索引 `idx` 加一；按 `0` 执行 `T = T[idx]` 并把索引清零；提交时检查 `T` 是否等于当前目标数字。因此，要选择某个节点的第 $i$ 个子元素，就要输入 $i$ 个 `1` 再输入一个 `0`。

对 `LOCK` 做深度优先搜索即可得到任意数字的按键路径。每个目标的路径依次拼接，形成最终密钥原文：

```python
def find_path(node, target, path=''):
    if node == target:
        return path
    if isinstance(node, list):
        for index, child in enumerate(node):
            result = find_path(child, target, path + '1' * index + '0')
            if result is not None:
                return result

key_text = ''.join(find_path(LOCK, target)
                   for target in range(1, 1338))
```

页面最后计算 `SHA-512(key_text)`，再与一个 64 字节常量逐字节异或。完整恢复代码为：

```python
from hashlib import sha512

ciphertext = bytes([
    62, 223, 233, 153, 37, 113, 79, 195, 9, 58, 83, 39, 245, 213,
    253, 138, 225, 232, 123, 90, 8, 98, 105, 1, 31, 198, 67, 83,
    41, 139, 118, 138, 252, 165, 214, 158, 116, 173, 174, 161, 6,
    233, 37, 35, 86, 7, 108, 223, 97, 251, 2, 245, 129, 118, 227,
    120, 26, 70, 40, 26, 183, 90, 172, 155,
])

digest = sha512(key_text.encode()).digest()
flag = bytes(a ^ b for a, b in zip(ciphertext, digest))
print(flag.decode())
```

得到：

```text
DUCTF{s3arch1ng_thr0ugh_an_arr4y_1s_n0t_th4t_h4rd_ab894d8dfea17}
```

## 方法总结

`0/1` 输入并不是二进制数，而是树遍历指令：连续的 `1` 表示子节点索引，`0` 表示下降一层。识别这一状态机后，1337 个目标只是可自动化的重复搜索。最终解密还必须使用所有路径按目标递增顺序拼接后的 SHA-512 摘要，不能分别哈希每段路径。
