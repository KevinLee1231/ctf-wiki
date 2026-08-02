# roundabout

## 题目简述

题目使用 `age` 的 passphrase 模式加密 flag。口令由《一桶白葡萄酒》中若干个随机单词直接拼接而成，并以图的形式给出提示：每个节点保存口令中的一个字符，真实口令的相邻字符之间都有边，同时混入约三倍数量的随机干扰边；节点和边顺序均被打乱。

因此目标是在有噪图中寻找一条覆盖所有节点一次的 Hamilton 路径，使路径字符能够分割成原文词表中的若干单词，并用候选口令实际解密 `secret.txt`。

## 解题过程

先从 `hint.txt` 解析 `id [letter];` 节点和 `id -- id;` 边，再从 `amontillado.txt` 提取小写单词集合。生成器中的 `random.random() % 2` 几乎总为真，所以它总把原始端点反向写出；官方解法解析时再交换一次端点，恢复为有向边。可先按节点字符多重集过滤词表：某个单词需要的任一字符数量超过图中数量时，它不可能出现在口令中。

搜索状态为“当前节点、已访问节点集合、当前字符串”。单纯枚举 Hamilton 路径分支过大，关键剪枝是用字符 Trie 判断当前字符串能否被切分为若干完整词，再接一个合法词前缀。只有仍可能形成词表串的状态才继续扩展。

```python
words = set(filtered_words)
prefixes = {""} | {word[:i] for word in words for i in range(1, len(word) + 1)}

def viable(text: str) -> bool:
    # text 必须能写成若干完整词，再接一个词前缀。
    reachable = {0}
    for start in range(len(text) + 1):
        if start not in reachable:
            continue
        if text[start:] in prefixes:
            return True
        for word in words:
            if text.startswith(word, start):
                reachable.add(start + len(word))
    return False

stack = [(node, frozenset([node]), node.letter) for node in nodes]
while stack:
    node, seen, text = stack.pop()
    if not viable(text):
        continue
    if len(seen) == len(nodes):
        try:
            print(age_decrypt(secret, text))
            break
        except Exception:
            continue
    for nxt in adjacency[node] - seen:
        stack.append((nxt, seen | {nxt}, text + nxt.letter))
```

仓库官方解法使用 `pygtrie` 实现同样的分词前缀检查，并在每条完整路径上调用 `pyrage.passphrase.decrypt`。AEAD 认证失败会自动排除错误口令，正确路径解出：

```text
tjctf{like_the_hit_indie_musical_2e8a47}
```

## 方法总结

- 这不是通用的 Hamilton 路径暴力题；自然语言词表是决定搜索能否完成的强剪枝条件。
- 节点字符是多重集，词表预过滤也必须按字符出现次数判断，不能只判断字符是否存在。
- 最终判据应使用 `age` 自带的认证解密，而不是只凭候选看起来像英语；认证标签为正确口令提供确定验证。
