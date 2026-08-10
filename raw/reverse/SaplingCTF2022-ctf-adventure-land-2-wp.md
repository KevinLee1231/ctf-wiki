# CTF Adventure Land 2

## 题目简述

从本题开始，目标不是提交文本，而是提交 replay.txt。服务器用原始游戏代码逐帧重放，只有在三个关卡中依次取得 flag 并完成游戏，才输出第二题 flag。可以修改本地代码分析地图，但提交的输入必须在未修改版本上成立。

## 解题过程

Input.py 把每帧按键编码为 5 位掩码：

~~~text
up=1, down=2, left=4, right=8, space=16
~~~

replay.txt 每行是一个五位二进制数，例如 01001 表示同时按 up 与 right。Map.py 从 map.png 中查找三个 PLAYER_SPAWN tile 和三个 FLAG tile；Game.gotFlag 在前两关触发 LevelTransition，最后一关把 won 置为真。

先用普通游戏模式完成一次全流程，保留 replay.txt，再用原程序检查：

~~~powershell
Get-Content replay.txt -Raw | python main.py check
~~~

官方仓库保留的 2023-fastest-without-tricks.replay 是一条不依赖漏洞的完整基线。提交任何能正常完成三关的有效 replay 后，Flag.py 无条件调用 printFlag(0,0)，输出：

~~~text
maple{c0ll3c71Ng_Fl4G5_1s_34sY_63ce2533dcfc1253}
~~~

## 方法总结

可重放游戏题的真实输入不是角色坐标，而是每帧按键状态。分析时应先读 replay 编码、胜利状态机和服务器 checker，再录制或程序化生成输入。修改本地地图只能帮助观察和规划；最终必须回到原始逻辑重放，避免提交只在魔改客户端中成立的路线。
