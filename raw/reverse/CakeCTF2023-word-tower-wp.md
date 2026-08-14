# Word Tower

## 题目简述

题目提供一个用 HSP3 制作的 Windows 字母拼词游戏。每关随机选择若干互不重复的动物名，把所有字母混洗到字母盒中；玩家必须使用全部字母，拼出与内部答案集合相符的单词。三关参数依次为：

- 第一关：从简单词表选 3 个词，限时 2 分钟；
- 第二关：从简单词表选 4 个词，限时 3 分钟；
- 第三关：从完整词表选 5 个词，最大长度提高到 15，限时只有 30 秒。

提交时程序要求每行左对齐、所有字母全部用完，并逐行检查提交词是否存在于本局 `answer` 数组。完成第三关后，状态常量变为 `0x223620`，程序才会解出并显示 flag。

官方解法不需要识别每一组乱序字母，而是用 Cheat Engine 修改运行时词表。核心障碍是理解游戏在内存中的答案表示和集合校验，而不是利用内存破坏漏洞，因此归入 `reverse`。

## 解题过程

### 关键观察

动物词表在进程内以固定步长 `0x70` 保存字符串。官方脚本先搜索一个确定存在的词 `sheep`，再按 `0x70` 向前回溯到词表开头，随后遍历整个词表。对每个动物名，脚本保留长度不变，却把所有字符改成同一个字符：

```lua
new_name = string.rep(string.char(0x61 + #name), #name)
```

因此长度为 $n$ 的动物名会统一变成由 $n$ 个 `chr(0x61+n)` 组成的字符串。例如长度 3 变成 `ddd`，长度 5 变成 `fffff`。游戏随后从已修改的词表复制答案并混洗字母，所以每种字母本身就泄露了对应单词长度。

更重要的是，提交检查只验证“每个提交词是否出现在答案数组中”，没有把已匹配答案标记为已使用。若本局抽到两个同长度、原本不同的动物名，它们被改写后会变成相同字符串，重复提交该统一字符串仍可通过。

### 修改词表

将 Cheat Engine 附加到 `wordtower.exe`，在其 Lua Engine 中运行官方脚本的核心流程：

```lua
local memscan = createMemScan()
local foundlist = createFoundList(memscan)

memscan.firstScan(
    soExactValue, vtString, rtTruncated,
    "sheep", nil, 0, 0x7fffffffffff, "+W*X",
    fsmNotAligned, "1", false, false, false, false
)
memscan.waitTillDone()
foundlist.initialize()

for i = 0, foundlist.Count - 1 do
    local base = tonumber(foundlist.getAddress(i), 16)

    -- 按 0x70 字节记录向前找到词表首项
    while #readString(base, 16, false) ~= 0 do
        base = base - 0x70
    end
    base = base + 0x70

    -- 用“长度对应字符”覆盖每一个词，字符串长度保持不变
    while true do
        local name = readString(base, 16, false)
        if #name == 0 then
            break
        end
        local replacement = string.rep(string.char(0x61 + #name), #name)
        writeString(base, replacement, false)
        base = base + 0x70
    end
end
```

重新开始关卡后，只需按字符分组：看到 `ddd` 就填一个长度 3 的答案，看到 `ffffff` 就填一个长度 6 的答案。按字母总数把每组拆成相应数量的同名答案，即可快速通过第三关的 30 秒限制。

### flag 生成与验证

进入 `GAMECLEAR` 后，程序先通过若干绘图状态值构造 38 字节数组，再从 `stage = 0x223620` 开始，对每个相邻四字节小端窗口执行异或，并在每轮把 key 增加 `0x3776`：

```python
key = 0x223620

for i in range(len(flag) - 4):
    value = int.from_bytes(flag[i:i + 4], "little")
    value ^= key
    key += 0x3776
    flag = flag[:i] + value.to_bytes(4, "little") + flag[i + 4:]
```

仓库中的 `challenge/flag.py` 用相同序列再次异或来验证构造结果，最终显示：

```text
CakeCTF{wow_you_know_a_lot_of_animals}
```

## 方法总结

- 核心技巧：定位固定步长的运行时词表，把每个候选词改写为“只编码长度”的规范形式，从而将限时字谜化为按字符分组。
- 识别信号：客户端本地持有完整答案；答案记录布局规则；游戏从该词表复制随机答案；校验只做集合成员判断，没有一一消费匹配项。
- 复用要点：游戏题不一定要逐题求解随机谜面。先找可被稳定定位的已知字符串，再检查相邻记录步长、答案复制时机和校验是否要求双射。修改时保持原字符串长度，可以避免破坏记录布局和后续内存。
