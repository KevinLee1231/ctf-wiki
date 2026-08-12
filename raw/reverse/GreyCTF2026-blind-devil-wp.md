# Blind Devil

## 题目简述

远端连续生成 100 个打乱的 $2\times2\times2$ 魔方，但不会显示贴纸位置。每次操作后只能获得一个 6 位观测：白、黄、绿、蓝、橙、红六种颜色的四张贴纸是否恰好占据某个完整物理面。所有转动，包括探测和撤销动作，都会计入步数；单个魔方上限为 25000 步。

这是一道按平均步数计分的题，没有静态 flag。服务在解完 100 个魔方后按平均步数计算积分并提交到 CTFd。虽然仓库把它放在 `ai` 下，但解法不依赖机器学习模型；关键是根据公开魔方转移规则和极度压缩的反馈重建隐藏状态，因此本文按 Reverse 归档。

## 解题过程

服务端 `STATE code=<0..63>` 的六位依次对应 `white, yellow, green, blue, orange, red`。一次 `move` 可发送标准 Singmaster 四分之一转动，另有 `V=U'`、`S=L'`、`G=F'` 三个别名。响应中的 `TRACE` 给出序列中每一步后的观测，因此可以把远端包装成如下 oracle：

```python
def observe_after(move_token):
    response = client.run_command(f"move {move_token}")
    return code_to_observation(response.trace_codes[-1])
```

直接枚举完整魔方状态并不可取。官方 group-theory 解法将每种颜色的四张贴纸视为一个集合，用公开转动规则预计算集合轨道和稳定子群，再在线执行保证覆盖这些轨道的短路径。

第一阶段在全转动群中寻找使一种颜色成为完整面的覆盖路径；获得已完成颜色后，后续阶段只使用保持现有颜色集合不变的稳定子群宏动作，再依次寻找新的完整颜色。默认计划以 `U`、`D`、`F`、`B` 四个集合为阶段目标：在合法 $2\times2\times2$ 转动群中，同时稳定这四组贴纸后只剩恒等状态，另外两面也随之完成。预计算只枚举贴纸集合轨道和较小的稳定子群，不需要展开全部 88179840 个合法状态。

实际服务还有一层歧义：观测只说明某种颜色形成了完整面，却不说明它落在哪一个固定面。官方解法用可撤销的单步探针判定已完成颜色所属的相对轴：

```text
Y 轴探针：U
Z 轴探针：F
X 轴探针：R
```

若转动后该颜色仍显示为完整面，就能把它归入相应对面轴；随后立即执行逆转恢复状态。每个轴仍有两个精确面的假设，解法逐一尝试这些假设，并在对应稳定子群中执行覆盖宏。只要宏动作端点破坏了此前已完成的颜色，就撤销该宏并淘汰该假设；若出现新完成颜色，则记录其轴并继续缩小状态集合。

在线控制逻辑可以概括为：

```python
while not all(observation.values()):
    preserved = frozenset(known_color_axes)
    for hypothesis in exact_face_hypotheses(known_color_axes):
        for chunk in subgroup_cover_chunks(hypothesis):
            apply_many(chunk)
            if solved():
                return
            if not all(observation[color] for color in preserved):
                apply_many(inverse(chunk))
                break
            if discovered_new_solved_color():
                classify_its_axis_with_undoable_probes()
                break
```

连接正式服务时先提交队伍 token，再用公开 `pow.py` 解 24 位前导零 PoW。参考客户端会对每个新 `CUBE` 重复上述流程，直到服务返回 `FINAL average=... score=...`。计分公式为：

$$
\operatorname{score}=\operatorname{round}\left(100+900\frac{\ln(25000/\bar m)}{\ln(25000/2000)}\right)
$$

其中 $\bar m\le 2000$ 时得 1000 分，$\bar m\ge 25000$ 时得 100 分。

## 方法总结

本题的决定性思路是把“看不见贴纸”转化为部分可观测状态机：公开转动规则提供确定的状态转移，6 位完成面掩码提供反馈。稳定子群保证新一阶段的搜索不会破坏已锁定结构；可撤销探针和假设淘汰则处理“知道颜色完成、却不知道位于哪个面”的歧义。所有探测都会计步，因此离线预计算轨道覆盖路径、在线只执行必要探针，是兼顾可解性与得分的关键。
