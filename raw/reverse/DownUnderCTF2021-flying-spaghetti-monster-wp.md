# DownUnderCTF 2021 - Flying Spaghetti Monster

## 题目简述

附件 `fsm.txt` 描述一个带自环的完全有向有限状态机。每行包含起点、终点、该边代表的 ASCII 码，以及一个仿射函数 $f_e(x)=a_ex+c_e$；其中 $a_e$ 为素数。服务不直接给出经过的字符，而是把路径上各边函数依次复合，只发送最终的一次函数 $F(x)=Ax+C$ 和终止状态，要求在严格时限内还原原字符串。

## 解题过程

对输入字符串的第一个字符，状态机先选择代表该字符的自环；此后每个字符选择一条从当前状态出发、边标签等于该字符 ASCII 码的边。沿路径按顺序执行：

$$
F_k=f_k\circ f_{k-1}\circ\cdots\circ f_1.
$$

若最后一条边为 $e$，此前的复合函数为 $G(x)$，则：

$$
F(x)=f_e(G(x))=a_eG(x)+c_e.
$$

所以从已知终止状态枚举入边时，只有满足 $C-c_e$ 可被 $a_e$ 整除的边才可能是最后一条边；剥去它后，前缀复合函数的常数项为 $(C-c_e)/a_e$，递归转到该边的起点。常数项回到 0 时，就回到了初始恒等函数 $x$。随机常数与每边不同的素数使错误分支通常会立即在整除检查处被剪掉。

```python
def peel(graph, state, constant):
    for previous, _, edge in graph.in_edges(state, data=True):
        poly = edge["f"]
        c_edge = int(poly.eval(0))
        a_edge = int(poly.eval(1) - poly.eval(0))

        delta = constant - c_edge
        if delta == 0:
            yield chr(edge["n"])
            return

        q, r = divmod(delta, a_edge)
        if r != 0:
            continue

        try:
            yield from peel(graph, previous, q)
        except ValueError:
            continue
        yield chr(edge["n"])
        return

    raise ValueError("no valid predecessor")
```

注意字符应在递归返回时追加：当前枚举的是路径末边，先恢复更早的边，最终才能得到正序明文。解析服务给出的 `F -> state` 后，只需取 $F$ 的常数项并调用上述回溯；系数 $A$ 的素因子也能辅助筛边，但对最后几轮的巨大整数做完整分解反而来不及，逐边执行一次 `divmod` 更快。

服务先给固定练习轮次，再给带 1～2 秒时限的随机长字符串，最后把包含 flag 的整段文本编码成同样的多项式。持续在线解析与回答，最终恢复出：

```text
DUCTF{fsms-m4k3-m3-smfh-fsmsfsmsfsmsfsmsfsm}
```

## 方法总结

本题的核心是恢复自定义状态机的编码语义，再逆序撤销仿射函数复合。已知终止状态把候选限制为入边，素数乘数和常数整除关系提供了廉价的路径判定。面对巨大多项式时，应利用结构做局部逆运算，而不是先对整体系数做昂贵的完整因式分解；同时必须把求解器接入实时交互，人工完成练习轮次并不足以通过最终时限。
