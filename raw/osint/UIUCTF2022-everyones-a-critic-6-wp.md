# Everyone's A Critic 6

## 题目简述

系列最后一题说明 Chuck 还藏着一条足以“永远暴露他”的信息。第五题已经确认目标 SteamID64 为 `76561199375368137`，因此这一题应继续枚举该 Steam 账号尚未检查的公开区域。提示指出 flag 正文以 `n0` 开头。

决定性证据位于 Counter-Strike 库存物品的自定义 Name Tag 中。只看物品名称或市场描述会得到“这不是刀”的语义提示，却看不到完整 flag；必须读取 Steam 库存接口返回的资产属性。

## 解题过程

### 1. 从已确认的 Steam 账号继续枚举

沿用第五题确定的 SteamID64，而不是再次依赖可修改的昵称。该 ID 已由前一题中的用户名、头像、位置和评测内容共同确认，本题不需要再次依靠一张孤立头像建立身份关系。

打开资料页的“库存”，选择 Counter-Strike 2（比赛时页面仍使用 CS:GO 名称）。当前公开库存只有一件武器皮肤：`MP9 | Slide (Minimal Wear)`，类型为 `Consumer Grade SMG`。

物品名称和类别已经能说明它是 SMG 而不是刀；单独的武器渲染图不显示自定义 Name Tag，因而不作为解题证据保留。仅凭物品类型仍不能严谨推出 flag 的精确大小写和符号。

### 2. 查询库存 API 的资产属性

Steam 的公开库存接口格式为：

```text
https://steamcommunity.com/inventory/<SteamID64>/<AppID>/<ContextID>?l=english&count=75
```

本题使用 SteamID64 `76561199375368137`、Counter-Strike AppID `730` 和普通库存 ContextID `2`：

```text
https://steamcommunity.com/inventory/76561199375368137/730/2?l=english&count=75
```

响应中的 `assets` 给出物品实例，`descriptions` 给出市场名称和类别，而真正的自定义标签位于与同一 `assetid` 关联的 `asset_properties`。精简后的关键字段如下：

```json
{
  "assetid": "26570909285",
  "name": "MP9 | Slide",
  "type": "Consumer Grade SMG",
  "asset_properties": [
    {
      "propertyid": 5,
      "string_value": "uiuctf{n0t_@_kn1f3!}",
      "name": "Name Tag"
    }
  ]
}
```

因此完整 flag 不是根据“MP9 不是刀”猜出的，而是直接来自该库存资产的 `Name Tag`：

```text
uiuctf{n0t_@_kn1f3!}
```

官方公开仓库的 [`osint/critic-6` 题目目录](https://github.com/sigpwny/UIUCTF-2022-Public/tree/main/osint/critic-6)还可用于核对提示和最终答案。即使 Steam 前端以后改变，SteamID64、AppID、ContextID、assetid 和上述字段关系已经足以复现证据链。

## 方法总结

系列题的最后一步是从账号识别转向平台对象枚举。确认 SteamID64 后，应系统检查游戏、评测、截图、好友、组和库存；库存物品还要区分通用商品描述与实例专属属性。Steam 自定义 Name Tag 属于后者，简单地只遍历 `descriptions` 会漏掉 flag，必须把 `assets` 与 `asset_properties` 按 `assetid` 对应起来。

最终 flag：

```text
uiuctf{n0t_@_kn1f3!}
```
