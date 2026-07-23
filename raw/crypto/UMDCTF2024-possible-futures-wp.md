# possible futures

## 题目简述

题目给出一个 `root.7z`，内部是一棵深度 10、约 35000 个文件的嵌套 7z 树。每个归档的密码都是其文件名的 MD5：

$$
\text{password}=\operatorname{MD5}(\text{filename}).
$$

深层会出现名为 `possible_flag_<n>.txt` 的文件，但绝大多数只含随机垃圾；只有极少数由固定概率分支写入真实 flag。题目同时提供生成器 `builder.py`，其中 Python `random` 使用固定种子，因此无需穷举解压整棵树，可以精确重演 PRNG 状态并预测含 flag 的分支。

## 解题过程

### 重演归档树

生成器固定使用：

```python
random.seed(
    "My name is Paul Muad'dib Atreides, duke of arrakis".encode()
)
```

复制 `ZipNode`、`get_new_int()` 和 `generate_random_tree()`，为节点增加
`parent` 引用，但不要真正创建归档。这样生成出的节点编号和目录拓扑与附件完全一致。

### 保持随机调用序列一致

生成器遍历每个候选文本时先调用一次 `random.random()` 判断是否写 flag。若未命中，还会调用
`random.randint()` 和 `random.choices()` 生成垃圾字符串。预测脚本即使不关心垃圾内容，也必须保留这些调用，否则后续 PRNG 状态会错位：

```python
def walk(node):
    for child in node.children:
        if child.filename.endswith(".txt"):
            if random.random() < 0.0005:
                print_path(child)
                return True

            # 结果不用保存，但必须消耗与 builder.py 相同的随机数。
            random_string(random.randint(5, 35))
        elif walk(child):
            return True
    return False
```

官方生成环境是 Python 3.12。为确保 `random.choices()` 等实现细节一致，应使用同一版本重演。

### 沿预测路径逐层解压

第一个命中节点是 `possible_flag_3968.txt`，从根到叶的归档链与密码如下：

| 归档 | MD5 文件名所得密码 |
|---|---|
| `root.7z` | `90cbb559e85c5e61bd39180a1ed10a54` |
| `future_number_3761.7z` | `81ad1418ccb7b9ca74ce6f4599bfdd48` |
| `future_number_3762.7z` | `6bf23d1186bad11c56c59518713b0c4c` |
| `future_number_3763.7z` | `a5c48564fe7a57028dd3e55e78b47eb0` |
| `future_number_3764.7z` | `790abec6791f365229a3fe0145e2aa7d` |
| `future_number_3765.7z` | `03fbdfc816aab8d0b6961996e4357cfd` |
| `future_number_3908.7z` | `8a2466a92aa8799d910d31bd3f0aeb7b` |
| `future_number_3953.7z` | `5f10d6b15ac51d92cc66b2722cfc9165` |
| `future_number_3967.7z` | `aecf605ce2d90410ea4fae817c32beff` |

按表从上到下解压，每次进入刚得到的下一层归档，最终读取
`possible_flag_3968.txt`：

```text
UMDCTF{th3_w@t3R_0F_l1fE_lE@ds_yOu_t0_v1cT0RY}
```

## 方法总结

- 核心技巧：利用公开生成器中的固定 PRNG 种子重演随机树和 flag 放置决策，只解压命中的一条路径。
- 识别信号：附件规模极大但同时给出生成脚本、固定 `random.seed()` 和确定性编号时，应优先模拟生成过程，而不是机械遍历所有归档。
- 复用要点：预测随机流程时，所有影响状态的调用都必须按原顺序保留；密码为文件名 MD5，只负责逐层解密，与预测哪个文本含 flag 是两个独立步骤。
