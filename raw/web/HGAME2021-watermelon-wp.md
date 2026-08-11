# watermelon

## 题目简述

附件是基于 Cocos 的合成小游戏，页面提示分数达到 2000 才能取得 flag。决定性逻辑不在游戏物理过程，而在游戏结束时调用的 `gameOverShowText` 函数：当分数超过 1999 时，函数对一段 Base64 字符串解码并弹窗显示结果。

## 解题过程

在浏览器开发者工具中切换到手机分辨率后运行游戏，并通过 Cocos 调试工具或源码搜索定位结束界面节点 `gameEndL`、`gameEndL1`。沿其调用链可以找到以下判分代码：

```javascript
gameOverShowText: function (score) {
    if (score > 1999) {
        alert(window.atob("aGdhbWV7ZG9feW91X2tub3dfY29jb3NfZ2FtZT99"));
    }
}
```

条件只控制是否执行 `window.atob()`，密文已经硬编码在客户端，因此不需要实际玩到 2000 分。直接在控制台执行解码：

```javascript
atob("aGdhbWV7ZG9feW91X2tub3dfY29jb3NfZ2FtZT99")
```

得到：

```text
hgame{do_you_know_cocos_game?}
```

## 方法总结

客户端游戏题首先应定位“胜利/失败后如何提交或显示结果”，而不是立刻修改分数或重玩流程。看到 Cocos 节点名和结束回调时，可沿 UI 节点到判分函数追踪；若 flag 或其可逆编码完全位于客户端，直接恢复常量即可，游戏分数只是触发条件。
