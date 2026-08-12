# Hackergame2020 超迷你的挖矿模拟器 WP

## 题目简述

后端以坐标和 `baseSeed` 动态生成近乎无限的地图。挖掘 `/damage` 时，程序先判断当前方块是否能被工具挖掘，等待 3 至 5 秒后才再次读取同一坐标并返回掉落物；`/reset` 可以在等待期间改变地图。最简解利用这个 TOCTOU 竞态，不需要完成官方预期的 48 位 Java PRNG 状态恢复。

## 解题过程

后端核心顺序可以概括为：

```java
if (location.getMaterial().harderThan(tool)) {
    waitFor(LONG_DURATION);
    return AIR;
} else {
    waitFor(SHORT_DURATION);
    return location.getMaterial();
}
```

`getMaterial()` 在判断前后都会根据当前 `baseSeed` 重新计算，而等待期间没有锁住地图版本。预期解要找一个“旧地图不是 FLAG、新地图是 FLAG”的坐标，再并发执行 damage 与 reset；官方还分析了 Java `Random` 的 48 位 LCG，并利用黑曜石坐标泄露将低 20 位和高 28 位分组爆破。

实际还有更短的非预期路径：固定的左上角 FLAG 被挖空后，该坐标先表现为空气。流程如下：

1. 对左上角 FLAG 坐标调用一次 `/damage`，使它进入已挖空集合。
2. 再对同一坐标发起 `/damage`。判定时该处是空气，因此走可挖分支并开始等待。
3. 在等待窗口内并发请求 `/reset`。重置清除挖空状态，左上角再次生成 FLAG。
4. 原 damage 请求醒来后再次调用 `getMaterial()`，读到新地图中的 FLAG，响应随掉落物返回当前用户的动态 flag。

请求必须真正并发，不能等 `/damage` 返回后再 reset。前端地图截图和两张梗图不提供竞态证据，因此没有归档。

## 方法总结

这是 API 层的检查—使用竞态：授权判断和实际取值之间存在可被另一个请求修改的共享状态。修复时应在同一事务或锁内完成检查与状态变更，或给地图版本加不可变快照并在提交阶段验证版本未变化。复杂的 PRNG 预测不应掩盖更直接的并发一致性问题。
