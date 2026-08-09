# Compress dot new

## 题目简述

附件把一棵函数式 Huffman 树序列化为 JSON，并在下一行给出由 `0`、`1` 组成的压缩位串。内部节点使用 `a`、`b` 保存左右子树，叶节点使用整数属性 `s` 保存字符码。目标是恢复树对应的前缀码并解码位串。

题目的决定性步骤是还原自定义数据结构与编码程序语义，因此归入 `reverse`。

## 解题过程

树节点只有两种形式：

- 若节点含有 `s`，它是叶节点，当前遍历路径就是该字符的 Huffman 码，字符值为 `chr(node["s"])`；
- 否则它是内部节点，进入 `a` 子树时追加 `0`，进入 `b` 子树时追加 `1`。

Huffman 码是前缀码。建立“比特路径到字符”的映射后，从左到右累计密文比特；当前累计串命中映射时输出对应字符并清空缓冲区。完整解码脚本如下：

```python
import json
from pathlib import Path
from typing import Any


path = Path(input("Path?> "))
tree_json, encoded = path.read_text(encoding="utf-8").splitlines()[:2]
tree = json.loads(tree_json)

codes: dict[str, str] = {}
queue: list[tuple[Any, str]] = [(tree, "")]

while queue:
    node, prefix = queue.pop(0)
    if "s" in node:
        codes[prefix] = chr(node["s"])
    else:
        queue.append((node["a"], prefix + "0"))
        queue.append((node["b"], prefix + "1"))

result: list[str] = []
current = ""
for bit in encoded.strip():
    current += bit
    if current in codes:
        result.append(codes[current])
        current = ""

if current:
    raise ValueError(f"末尾存在无法解码的比特串: {current!r}")

print("".join(result))
```

官方总题解未记录最终输出文本，但上述脚本已经包含从附件两行数据到明文的完整路径；运行时还额外检查末尾是否存在不能匹配任何叶节点的残余比特，避免静默接受截断输入。

## 方法总结

- 核心技巧：从 JSON 节点字段恢复二叉前缀码树，再按路径 `a=0`、`b=1` 解码位流。
- 识别信号：递归结构中内部节点保存两个子节点、叶节点保存字符，而附件另有纯二进制字符串时，应联想到 Huffman 或其他二叉前缀编码。
- 复用要点：遍历树既可递归也可使用队列；真正的正确性条件是每次命中叶节点后重置缓冲区，并保证最终没有残余比特。
