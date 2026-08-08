# MiniLCTF2020 - MiniGameHacking

## 题目简述

附件是 Unity 游戏。可直接从 `data.unity3d` 的字符串区恢复 flag，也可以按预期路线修改托管逻辑，让角色不再死亡并通关 15 个关卡。决定性工作是分析 Unity 资源和 `Assembly-CSharp.dll`，因此归入 Reverse。

## 解题过程

最快的证据恢复方式是对游戏数据做字符串搜索。原参赛记录在 `data.unity3d` 尾部找到：

```text
minil{diosamasayikou}
```

若按游戏逻辑复现，先从 Unity 的 Managed 目录提取 `Assembly-CSharp.dll`，用 dnSpy/ILSpy 搜索伤害和死亡处理。关键方法近似为：

```csharp
public void OnDamaged() {
    if (this.isDead) return;
    this.isDead = true;
    Object obj = GameObject.FindGameObjectWithTag("Player");
    Camera.main.SendMessage("ShakeHandler", 10f);
    this.damuManager.SendMessage("RestartRound");
    this.bossHealthManager.SendMessage("FullHealth");
    Object.Destroy(obj);
    base.Invoke("RespawnPlayer", 1f);
}
```

移除 `RestartRound`、`Object.Destroy(obj)` 等失败路径，或让方法开头直接返回，再保存程序集并替换原 DLL。角色不会因受伤重置，完成全部关卡后同样显示 flag。

当前官方 Git 仓库没有保留该题二进制，以上方法由多份参赛记录中一致的 `data.unity3d` 结果和反编译方法交叉确认；没有伪造缺失的完整程序集偏移。

## 方法总结

Unity 题先检查资源、序列化数据和 Managed 程序集。直接字符串命中可作为结果证据，但还应说明预期机制：定位状态变化函数，修改死亡、扣血或关卡重置分支。补丁应尽量小，避免破坏场景加载和通关触发器。
