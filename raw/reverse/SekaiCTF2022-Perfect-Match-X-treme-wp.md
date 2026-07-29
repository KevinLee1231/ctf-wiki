# Perfect Match X-treme

## 题目简述

题目附件是一款仿《Fall Guys》Perfect Match 关卡的 Unity 游戏。正常玩法是记住地砖上的水果，并在倒计时结束前站到指定水果上；但第三轮指定的是永远不会生成到地砖上的 kiwi，因此正常通关并不可行。

真正的目标是逆向 Unity 的 C# 逻辑与场景资源：绕过第三轮的必败状态，使胜利界面显示原本隐藏的 12 段 TextMeshPro 文本，再按界面顺序拼出 flag。

## 解题过程

附件使用 Mono 后端，核心逻辑位于 `PerfectMatch_Data/Managed/Assembly-CSharp.dll`。用 dnSpy、ILSpy 等 .NET 反编译器打开后，可以定位到 `GameManager`、`Grid` 和 `UI` 三个类。

`GameManager.PopulateFruitDictionary()` 注册了 7 种水果：

```csharp
fruitDictionary.Add(1, Resources.Load<Sprite>("Images/apple"));
fruitDictionary.Add(2, Resources.Load<Sprite>("Images/banana"));
fruitDictionary.Add(3, Resources.Load<Sprite>("Images/cherry"));
fruitDictionary.Add(4, Resources.Load<Sprite>("Images/orange"));
fruitDictionary.Add(5, Resources.Load<Sprite>("Images/watermelon"));
fruitDictionary.Add(6, Resources.Load<Sprite>("Images/grape"));
fruitDictionary.Add(7, Resources.Load<Sprite>("Images/kiwi"));
```

然而生成地砖时调用的是：

```csharp
int randomFruitPosition = Random.Range(1, fruitDictionary.Count);
```

Unity 的整数版 `Random.Range(min, max)` 不包含上界。`fruitDictionary.Count` 为 7，所以这里只会产生 1 到 6，kiwi 永远不会进入 `chosenFruitHashSet`，也不会出现在任何地砖上。第三轮却被硬编码为：

```csharp
if (roundNumber == MAX_ROUND_NUMBER)
{
    chosenFruit = fruitDictionary[7];
    return;
}
```

`Grid.RemoveIncorrectTiles()` 会保留名称包含 `chosenFruit.name` 的地砖。第三轮指定 kiwi 后没有任何地砖符合条件，玩家必然坠落。这不是操作技巧问题，而是程序逻辑故意制造的不可达胜利状态。

继续检查结束流程。三轮结束后，`Grid` 最终调用：

```csharp
FindObjectOfType<UI>().SetGameState(!gameManager.isEliminated);
```

而 `UI.SetGameState(true)` 会显示胜利提示，并依次启用 `text11` 至 `text33` 的 12 个 TextMeshPro 对象。可以在 dnSpy 中修改 `IncreaseRound()`，让第三轮结束后直接进入胜利状态；也可以把 `Grid` 中的参数改成恒为 `true`。保存程序集并重新启动游戏后，胜利界面会显示全部 flag 片段。

若不运行游戏，也可以直接检查 `PerfectMatch_Data/level0` 场景资源。按 `text11`、`text12`、……、`text33` 的界面顺序读取文本，可恢复出：

```text
SEKAI{F4LL_GUY5_
H3CK_15_
1LL3G4L}
```

拼接得到：

```text
SEKAI{F4LL_GUY5_H3CK_15_1LL3G4L}
```

题目 README 的官方玩法视频只用于说明原版 Perfect Match 的记忆地砖规则；关键漏洞、胜利条件和 flag 均可由附件中的程序集与场景资源独立确认，因此正文不依赖该视频。

## 方法总结

这道题的关键不是硬玩小游戏，而是同时检查“随机数据的取值范围”和“胜利界面的资源状态”。整数版 `Random.Range` 的上界不包含在结果中，导致第三轮目标水果与实际地砖集合不相交；修改胜负分支后，隐藏的 TextMeshPro 对象就会直接给出 flag。

分析 Unity Mono 游戏时，可以优先查看 `Managed/Assembly-CSharp.dll` 恢复业务逻辑，再结合场景文件核对 UI 文本。代码解释控制流，资源文件则能补齐没有直接写在程序集字符串中的最终内容。
