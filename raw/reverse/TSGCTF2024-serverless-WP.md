# TSGCTF2024 serverless

## 题目简述

附件是一份 Envoy `compose.yml`，其中配置了数百条内部重定向规则。请求路径必须形如 `/TSGCTF%7B...%7D`；Envoy 会反复对路径做正则替换，最多允许 2000 次内部重定向。路径最终变成 `/` 时返回 `ok`，其他无法继续匹配的路径返回 `ng`。

题目随附的单页 PDF 把全部规则画成一张极宽的有向图：少量关键路径混在大量干扰边中。视觉对照确认了图的用途，但真正求解应从 `compose.yml` 的可搜索规则直接建图，避免人工沿超大图追线。

## 解题过程

### 1. 先确定 flag 的整体格式

规则中有 26 条成对删除规则：

```text
A(a) -> ""
B(b) -> ""
...
Z(z) -> ""
```

可把大写字母视为一种左括号，把对应的 `(小写字母)` 视为右括号。没有其他规则能直接删除 flag 前缀 `TSGCTF`，所以在处理完花括号内容后，路径必须形成：

```text
TSGCTF(f)(t)(c)(g)(s)(t)
```

从右向左依次消去 `F(f)`、`T(t)`、`C(c)`、`G(g)`、`S(s)`、`T(t)`，最终清空前缀。

与 `{`、`}`、`_` 和游标 `/` 有关的规则只有：

```text
}{ -> +
}  -> )
{  -> (/
_  -> )(/
/) -> )
```

由于没有规则生成新的花括号，也没有规则删除 `+`，可行 flag 必须由六段组成：

```text
TSGCTF{part1_part2_part3_part4_part5_part6}
```

应用结构规则后变为：

```text
TSGCTF(/part1)(/part2)(/part3)(/part4)(/part5)(/part6)
```

六段分别需要生成 `f、t、c、g、s、t`。

### 2. 把游标重写规则抽象为有向图

其余关键规则都让 `/` 向右移动一到两个输入字符。根据 substitution 长度可分为三类：

1. 起始边：`/ab -> kXYZ/`。读取标签 `ab`，产生目标小写字母 `k`，并把状态设为三字母 `XYZ`。
2. 中间边：`/ab -> (s)(t)(u)XYZ/`。它对应从状态 `UTS` 转移到 `XYZ`；产生的 `(s)(t)(u)` 会与当前大写状态配对消去。
3. 终止边：`/ab -> (s)(t)(u)/`。从状态 `UTS` 转移到空状态。

于是规则可表示为带输入标签的有向边。为了方便从任意起始字母寻找可终止路径，solver 建立反向邻接表，并从空状态做一次 BFS：

```python
edges_to[target].append((source, label))

parent = {}
queue = deque([""])
while queue:
    node = queue.popleft()
    for child, label in edges_to[node]:
        if child in parent:
            continue
        parent[child] = (node, label)
        queue.append(child)
```

这里一张 BFS 树即可同时回答六个起始字母到终态的路径，不需要在数百条规则中逐条试请求。

### 3. 按 `f、t、c、g、s、t` 重建六段

以第一段 `f` 为例，唯一有效路径为：

```text
f --ht--> VDM --tp--> EJW --r--> VJV --d1--> YZA
  --re--> ZAA --ct--> EHH --5---> WSM --ca--> HCB --n--> ""
```

连接边标签得到：

```text
http-red1rect5-can
```

对 `reversed("TSGCTF".lower())`，即 `f、t、c、g、s、t`，分别沿 BFS 父指针走到空状态并连接标签：

```python
parts = []
for initiator in reversed("TSGCTF".lower()):
    labels = []
    node = initiator
    while node != "":
        node, label = parent[node]
        labels.append(label)
    parts.append("".join(labels))

flag = "TSGCTF{" + "_".join(parts) + "}"
```

最终得到：

```text
TSGCTF{http-red1rect5-can_funct10n_as-tur1ng-c0mp1ete_m4rk0v-a1g0rithm_proce55ing_funct10n}
```

把该路径交给规则系统后，六段分别化为 `(f)(t)(c)(g)(s)(t)`，继而与 `TSGCTF` 成对消去，路径变为 `/` 并返回 `ok`。

## 方法总结

本题把 Envoy 的内部重定向规则当作字符串重写系统。先用不可生成/不可删除字符推导六段 flag 结构，再把游标 `/` 后的局部替换抽象为有限状态图，最后从终止状态反向 BFS 求出每个起始字母对应的输入标签串。虽然官方生成器把这一系统描述为带大量干扰边的图，但求解时只需保留“起点、三字母状态、终点”三类节点语义；这也是从巨型配置中提炼自动机模型的关键。
