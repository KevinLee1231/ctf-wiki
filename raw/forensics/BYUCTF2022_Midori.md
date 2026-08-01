# BYUCTF 2022 - Midori

## 题目简述

附件 `midori.zip` 内含日文版 Pokémon Green 的 512 KiB Game Boy ROM `midori.gb` 和 32 KiB 存档 `midori.sav`。题目要求把存档中刻意设置的日文名称转写为拉丁字母 flag。

## 解题过程

一种做法是用支持 Game Boy 存档的模拟器加载 ROM 与 SAV，查看主角名称及当前队伍；也可以用 Pokémon 一代存档编辑器直接解析名称字段，避免完整游玩。关键证据不在 ROM 的普通剧情里，而在题目已经准备好的存档状态。

主角和队伍名称依次拼出：

```text
ポケモン ミドり やった こと が あるか
```

按日语罗马字逐词转写：

```text
pokemon midori yatta koto ga aruka
```

将空格改为下划线并套用赛事格式：

```text
byuctf{pokemon_midori_yatta_koto_ga_aruka}
```

当前仓库中的 ROM 与 SAV 均可直接解压验证；无需依赖官方旧题解中列举的特定模拟器或在线服务。

## 方法总结

对于游戏取证题，应先判断答案来自程序逻辑还是已有存档状态。本题的决定性证据是存档内的角色/队伍名称；解析存档和在模拟器中查看只是两种等价取证路径，最后还需按题目指定规则做日文罗马字转写。
