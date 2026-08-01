# GlacierCTF2022 - ClipRip

## 题目简述

剪贴板管理器保存一条未脱敏的密码和大量重复历史，但远程接口每次最多列出最近 100 条。最前方还有第二条已脱敏密码。题目不需要破坏内存，而是利用 `restore` 对 deque 的更新语义逐步删除或轮换历史，使第一条 flag 进入可见窗口。

## 解题过程

`restore(index, false)` 先复制目标条目，再从原位置删除它：

```cpp
const ClipboardEntry entry = entries_[index];
entries_.erase(std::next(entries_.begin(), index));

if (entries_.empty() || entry != entries_[0])
    entries_.push_front(entry);
```

若目标与当前第 0 条不同，它会被移动到最前方；若目标内容与当前条目完全相同，比较条件为假，目标副本被删除后不会重新插入。初始化脚本把同一组 Simpsons chalkboard 文本重复写入六轮，因此历史中存在大量可用于后一种情况的重复项。

自动化脚本每轮执行 `list -l 100`，去掉输出中的索引后比较内容。若当前条目在后面再次出现，就对那个索引执行 `restore`，删除重复副本；没有可删除的同值项时，则轮换一个普通索引到队首，换一个新的“当前值”继续消除。核心循环可写成：

```python
lines = list_entries(100)
current = lines[0]

for index, value in enumerate(lines[1:], 1):
    if value == current:
        restore(index)
```

随着重复历史不断缩短，最早保存且未标记为 `redact_` 的密码进入最近 100 条。发现 `glacierctf` 后将该项恢复到索引 0，再执行 `list -l 1`，得到：

```text
glacierctf{s1mps0n_1ntr0s_st1ll_h1t_h4rd}
```

## 方法总结

“恢复历史”是有状态操作，安全性不能只看显示时是否截断。这里的复制、删除和相等性去重组合成了一个可控的历史压缩原语；重复条目让攻击者跨越 100 条窗口。由于主线是还原容器状态变化并构造操作序列，而非越界写或控制流劫持，本题归 Reverse。
