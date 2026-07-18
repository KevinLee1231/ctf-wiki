# SUCTF2026-easygal

## 题目简述

题目是 Unity IL2CPP 打包的 galgame。玩家必须依次走完 60 个剧情节点，每个节点二选一；每个选项带有 `weight`、`value`、`flag` 和 `marker`。结算时只有总 `weight <= 132` 且总 `value == 322` 才进入真结局，真结局再将所选选项的 `marker` 按顺序拼接并计算 MD5，生成 `SUCTF{...}`。

直接枚举共有 `2^60` 条路线，不可行。题目数据把真结局设计为唯一最优路径，适合用带路径计数和恢复的分阶段背包 DP 求解。

## 解题过程

### 1. 从 IL2CPP 程序定位剧情数据与结算逻辑

用 Il2CppDumper 一类工具将 `GameAssembly.dll` 与 `global-metadata.dat` 配对恢复类型和方法名后，关键类是 `GameManager`、`GameStateStore`、`FlagUtility` 与 `StoryDatabase`。

`GameManager.LoadStory()` 通过 `Resources.Load<TextAsset>(...)` 读取 `story` TextAsset，再用 `JsonUtility.FromJson<StoryDatabase>()` 反序列化。当前 `story.json` 的元数据为：

```json
{
  "maxWeight": 132,
  "trueEndingValue": 322,
  "nodeCount": 60,
  "verificationMethod": "DP count exact optimum paths"
}
```

选项点击后的状态更新为：

```text
currentWeight += choice.weight
currentValue  += choice.value
flags.Add(choice.flag)       # HashSet，仅记录
markers.Add(choice.marker)   # List，保留选择顺序
```

60 个节点全部走完后，源码按以下顺序结算：

```text
currentWeight > maxWeight              -> Failure
currentWeight <= maxWeight
  且 currentValue == trueEndingValue   -> True
其它                                     -> Normal
```

这里有两个容易误读的点：

- `flags` 集合不参与真结局判定，也不参与最终哈希；
- 程序本身只检查 `value == 322`，并不检查“最大值”或“路径唯一”。`322` 恰好是全局最大值且只有一条路线达到它，是当前数据集经独立 DP 验证得到的性质。

`FlagUtility.BuildTrueEndingFlag()` 会跳过空 marker，按原顺序拼接其 UTF-8 字节，计算小写十六进制 MD5，再包成 `SUCTF{digest}`。

### 2. 提取 `story.json`

最稳妥的方法是从 Unity 的 `resources.assets` 导出名为 `story` 的 TextAsset。总 PDF 给出的当前实例还可以直接按资源名和长度字段取出 JSON：

```python
import json
from pathlib import Path


def extract_story(resources_assets):
    data = Path(resources_assets).read_bytes()
    marker = b"story\x00\x00\x00"
    offset = data.find(marker)
    if offset < 0:
        raise RuntimeError("未找到 story TextAsset")
    length = int.from_bytes(data[offset + 8:offset + 12], "little")
    payload = data[offset + 12:offset + 12 + length]
    return json.loads(payload.decode("utf-8"))
```

若不同 Unity 版本的序列化布局使该固定偏移失效，应改用 Unity 资源解析工具导出 TextAsset，而不是继续猜长度字段。官方源码仓库中也直接提供了 `Assets/Resources/Story/story.json`，可用于核对附件提取结果。

### 3. 用 DP 同时求最优值、路径数和唯一路径

设 `dp[w]` 表示处理完当前若干节点、总重量恰为 `w` 时的最优状态。状态保存：

```text
value    该重量下最大价值
count    达到该最大价值的路径数
path     仅当 count == 1 时保留的选择序列
markers  仅当 count == 1 时保留的 marker 串
```

每个节点必须选一个选项，因此每轮都从旧 `dp` 构造全新的 `next_dp`。同一重量出现更大价值时覆盖；价值相同时累加路径数并丢弃代表路径，避免把“随便保存一条”误写成“答案唯一”。

```python
import hashlib
import json
from pathlib import Path


def solve_story(story):
    max_weight = int(story["meta"]["maxWeight"])
    target_value = int(story["meta"]["trueEndingValue"])

    # weight -> (best_value, path_count, path_flags, marker_string)
    dp = {0: (0, 1, [], "")}

    for node in story["nodes"]:
        next_dp = {}
        for weight, (value, count, path, markers) in dp.items():
            for choice in node["choices"]:
                new_weight = weight + int(choice["weight"])
                if new_weight > max_weight:
                    continue

                new_value = value + int(choice["value"])
                candidate_path = path + [choice["flag"]] if count == 1 else None
                candidate_markers = markers + choice["marker"] if count == 1 else None
                previous = next_dp.get(new_weight)

                if previous is None or new_value > previous[0]:
                    next_dp[new_weight] = (
                        new_value,
                        count,
                        candidate_path,
                        candidate_markers,
                    )
                elif new_value == previous[0]:
                    next_dp[new_weight] = (
                        new_value,
                        previous[1] + count,
                        None,
                        None,
                    )
        dp = next_dp

    best_value = max(value for value, _, _, _ in dp.values())
    optimal = [
        (weight, state)
        for weight, state in dp.items()
        if state[0] == best_value
    ]
    optimal_count = sum(state[1] for _, state in optimal)

    if best_value != target_value:
        raise RuntimeError(
            f"配置目标 {target_value} 与 DP 最大值 {best_value} 不一致"
        )
    if optimal_count != 1:
        raise RuntimeError(f"最优路径不唯一：{optimal_count}")

    weight, (_, _, path, markers) = optimal[0]
    digest = hashlib.md5(markers.encode("utf-8")).hexdigest()
    return weight, path, markers, f"SUCTF{{{digest}}}"


story = json.loads(Path("story.json").read_text(encoding="utf-8"))
weight, path, markers, flag = solve_story(story)
print("OptimalWeight=", weight)
print("UniquePath=", ",".join(path))
print("MarkerString=", markers)
print("Flag=", flag)
```

运行结果为：

```text
OptimalWeight=132
BestValue=322
OptimalPathCount=1
UniquePath=N1B,N2B,N3A,N4B,N5A,N6A,N7B,N8A,N9A,N10A,
N11A,N12A,N13A,N14A,N15B,N16B,N17A,N18B,N19A,N20A,
N21A,N22A,N23B,N24B,N25B,N26B,N27A,N28B,N29B,N30B,
N31B,N32B,N33B,N34B,N35B,N36A,N37A,N38A,N39B,N40A,
N41A,N42A,N43B,N44A,N45B,N46A,N47A,N48A,N49B,N50B,
N51B,N52A,N53B,N54B,N55B,N56B,N57A,N58A,N59A,N60B
```

按这条唯一路径拼接 marker 并计算 MD5，得到：

```text
SUCTF{92d1c2c3f6e55fabbc3a6ffde57c7341}
```

官方 WP 中指向作者本机 `E:/Unity down/...` 的绝对路径只是源码位置提示，其他环境无法访问；其关键信息已在正文中完整展开，因此不保留这些失效链接。

## 方法总结

- Unity IL2CPP 题先恢复类型/方法，再追 `Resources.Load`、反序列化对象和最终结算函数；不要靠手玩剧情猜条件。
- 60 次二选一共有 `2^60` 条路线，应按“阶段 + 总重量”做 DP，而不是爆搜。
- DP 若需要证明唯一性，必须在并列最优时累加路径数；保存一条代表路径并不能证明只有一条路径。
- 区分程序显式条件与数据集性质：代码只要求 `value == 322`，而“322 是唯一全局最优”来自对当前 `story.json` 的验证。
- 最终 flag 使用有序 marker 列表，不使用无序的 flags 集合；容器类型会直接影响哈希输入顺序。
