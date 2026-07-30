# NepCTF2026 【Game】共生（本地） Writeup

## 题目简述

附件是 Unity IL2CPP 游戏。玩家需要在本地关卡中触发规定事件，再依据运行状态和个人 team token 计算 flag。决定性障碍是从 `GameAssembly.dll` 与 IL2CPP metadata 中恢复事件状态机，而不是修改存档或直接搜索 flag 字符串。

主要分析对象为：

```text
GameAssembly.dll
J07_Data/il2cpp_data/Metadata/global-metadata.dat
J07_Data/data.unity3d
```

## 解题过程

使用 Il2CppDumper/Il2CppInspector 恢复类型和方法映射后，重点关注 `CampaignRunState` 及关卡中的 `StorePoint`、`WinPoint`、`TeleportPoint`、`TeamTokenPopup`。状态初值为：

```text
A = 0x243F6A8885A308D3
B = 0x13198A2E03707344
count = 0
```

按正常通关路线，事件序列为：

```text
0x100, 0x200, 0x110, 0x111, 0x112, 0x201,
0x120, 0x121, 0x122, 0x124, 0x125, 0x202
```

序列中没有 `0x123`。Scene3 的传送点会把角色从 `x = 135.6` 直接送到 `x = 185.4`，越过位于 `x = 176.1` 的检查点，所以按实际关卡路线只能触发 12 个事件。

每次事件按如下方式更新两个 64 位状态，所有运算均模 $2^{64}$：

```python
MASK64 = (1 << 64) - 1

def rol64(x, n):
    n &= 63
    return ((x << n) | (x >> (64 - n))) & MASK64

def mix(a, b, count, event):
    event &= 0xffffffff
    count += 1

    a ^= (event + 0x9E3779B97F4A7C15 +
          ((a << 6) & MASK64) + (a >> 2)) & MASK64
    a = rol64(a, event % 23 + 7)
    a = (a * 0xD6E8FEB86659FD93) & MASK64

    b = (b + ((event * 0xA0761D6478BD642F) ^ a)) & MASK64
    b = rol64(b, count % 29 + 11)
    b = (b * 0xE7037ED1A0B428DB) & MASK64
    return a, b, count
```

依次混合 12 个事件后得到：

```text
A = 9383f49cdcbaef22
B = 408c3aecdb630a39
count = 12
```

最终原文格式为：

```text
j07-local-v2|<team token>|9383f49cdcbaef22|408c3aecdb630a39|12
```

计算其 SHA-256，并套入比赛 flag 格式：

```python
from hashlib import sha256

team_token = "替换为本队 token"
preimage = (
    f"j07-local-v2|{team_token}|"
    "9383f49cdcbaef22|408c3aecdb630a39|12"
)
flag = f"NepCTF{{{sha256(preimage.encode()).hexdigest()}}}"
print(flag)
```

由于 team token 不同，每支队伍得到的最终散列也不同。

## 方法总结

本地部分考查 Unity IL2CPP 类型恢复、场景路线核对和状态机复现。最容易出错的地方是凭静态对象列表把 `0x123` 也加入事件序列；应将反编译代码与实际传送坐标交叉验证。状态更新还必须逐步截断为 64 位，并保留 16 位十六进制宽度，否则最终 SHA-256 原文会与程序不一致。
