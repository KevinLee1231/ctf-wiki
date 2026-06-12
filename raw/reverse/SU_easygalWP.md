# SU_easygal

## 题目简述
题目是 Unity/IL2CPP galgame 逆向。题干说明需要在 60 个剧情节点中不断选择，唯一正确路线对应真正结局；核心数据在 Unity 资源中反序列化为 `Story` 结构，而不是靠人工试剧情分支。

每个节点包含多个 choice，每个 choice 带 `weight/value/flag/marker`。完成 60 个节点后，程序要求总 `weight` 不超过阈值且总 `value` 精确等于目标值，再把所选路径的 `marker` 串接后取 MD5 作为答案。解题本质是从 IL2CPP/资源中恢复 Story 数据，并把选择过程建模为带路径恢复的背包/DP。

## 解题过程
IL2CPP 打包程序可先用 Il2CppDumper 恢复符号，再用 IDA 打开 `GameAssembly.dll`。关键逻辑是加载剧情数据并反序列化为 `Story` 对象；如果加载失败，程序会使用内置默认剧情数据。

逆向后可以把核心数据结构概括成：

```text
Story.meta.maxWeight = 132
Story.meta.targetValue = 322
Story.nodes.length = 60

Node.choices[]:
  weight: 本次选择消耗
  value:  本次选择得分
  flag:   用于记录路径
  marker: 用于最终 MD5
```

每做一个选择，程序会把 `weight/value` 累加，把 `flag` 放入集合，把 `marker` 按顺序放入列表。60 个节点全部选择完成后，要求总 `weight <= 132` 且总 `value == 322`；最终把所有 `marker` 拼接后取 MD5 得到答案。因此这题可以直接建模为带路径恢复的背包/DP。

核心求解脚本如下：

```python
import csv
import hashlib
import json
import sys
from pathlib import Path


def extract_story_json(resources_assets: Path) -> dict:
    data = resources_assets.read_bytes()
    needle = b"story\x00\x00\x00"
    idx = data.find(needle)
    if idx < 0:
        raise RuntimeError("未找到名为 story 的嵌入资源")

    length = int.from_bytes(data[idx + 8:idx + 12], "little")
    json_bytes = data[idx + 12:idx + 12 + length]
    return json.loads(json_bytes.decode("utf-8"))


def solve_story(story: dict) -> dict:
    max_weight = int(story["meta"]["maxWeight"])

    # dp[当前总重量] = (最大价值, 达到该最大价值的路径数, 一条代表路径)
    dp = {0: (0, 1, [])}
    for node in story["nodes"]:
        ndp = {}
        for cur_w, (cur_v, cur_count, cur_path) in dp.items():
            for choice in node["choices"]:
                nw = cur_w + int(choice["weight"])
                if nw > max_weight:
                    continue
                nv = cur_v + int(choice["value"])
                npath = cur_path + [choice]

                if nw not in ndp or nv > ndp[nw][0]:
                    ndp[nw] = (nv, cur_count, npath)
                elif nv == ndp[nw][0]:
                    ndp[nw] = (nv, ndp[nw][1] + cur_count, ndp[nw][2])
        dp = ndp

    best_value = max(v for v, _, _ in dp.values())
    best_weights = [w for w, (v, _, _) in dp.items() if v == best_value]
    best_count = sum(c for _, (v, c, _) in dp.items() if v == best_value)

    chosen_weight = best_weights[0]
    chosen_path = next(
        path for w, (v, _, path) in dp.items()
        if w == chosen_weight and v == best_value
    )

    markers = [choice["marker"] for choice in chosen_path]
    marker_string = "".join(markers)
    final_flag = f"SUCTF{{{hashlib.md5(marker_string.encode()).hexdigest()}}}"

    return {
        "meta": story["meta"],
        "optimal_weight": chosen_weight,
        "optimal_value": best_value,
        "optimal_path_count": best_count,
        "chosen_flags": [choice["flag"] for choice in chosen_path],
        "markers": markers,
        "marker_string": marker_string,
        "final_flag": final_flag,
        "path": chosen_path,
    }


def write_optimal_csv(story: dict, solved: dict, out_csv: Path) -> None:
    with out_csv.open("w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow([
            "node", "chosen", "weight", "value", "flag",
            "marker", "dayLabel", "speaker", "choice_text",
        ])
        for i, (node, choice) in enumerate(zip(story["nodes"], solved["path"]), 1):
            w.writerow([
                i,
                choice["flag"][-1],
                choice["weight"],
                choice["value"],
                choice["flag"],
                choice["marker"],
                node["dayLabel"],
                node["speaker"],
                choice["text"],
            ])


def main() -> None:
    resources_assets = Path(sys.argv[1])
    outdir = Path(sys.argv[2]) if len(sys.argv) == 3 else Path.cwd()
    outdir.mkdir(parents=True, exist_ok=True)

    story = extract_story_json(resources_assets)
    solved = solve_story(story)
    write_optimal_csv(story, solved, outdir / "optimal_path.csv")

    print("meta =", json.dumps(story["meta"], ensure_ascii=False))
    print("optimal_weight =", solved["optimal_weight"])
    print("optimal_value =", solved["optimal_value"])
    print("optimal_count =", solved["optimal_path_count"])
    print("marker_string =", solved["marker_string"])
    print("final_flag =", solved["final_flag"])


if __name__ == "__main__":
    main()
```

## 方法总结
- 核心技巧：IL2CPP 数据恢复 + DP/背包
- 识别信号：剧情选择节点有权重和收益约束，最终校验是总和条件。
- 复用要点：恢复 Story 数据结构后，把选择问题建模为动态规划/背包，求满足约束的 marker 序列。
