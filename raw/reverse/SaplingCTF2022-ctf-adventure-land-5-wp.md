# CTF Adventure Land 5

## 题目简述

第五题要求严格小于 1285 帧。除前面的路线、尖刺和假墙优化外，还要利用关卡切换逻辑：取得前两关 flag 后，LevelTransition 播放 120 帧淡入淡出，但期间没有冻结玩家；第 65 帧又把一个固定向量加到玩家当前坐标。切换期间提前移动，会把位移带入下一关出生点。

## 解题过程

LevelTransition.tick 的核心是：

~~~python
if self.ticks == 65:
    x, y = self.teleportVectors[self.game.level]
    self.game.player.x += tileToWorldCoords(x)
    self.game.player.y += tileToWorldCoords(y)
    self.game.level += 1
~~~

这个向量按“玩家仍停在 flag 处”设计，但主循环在 transition 存在时仍处理输入和 Player.tick。于是拿到 flag 后不要空等：在第 65 帧前向下一段路线方向持续移动。固定向量最终叠加到已经偏移的坐标，使角色进入下一关时比正常出生点更靠前；两次切换都可节省帧数。

将 transition 移动加入 1399 帧路线，官方 1279-tele-bug-wall-hack-fast-track-walk-spikes.replay 用 1279 帧完成，满足 $1279<1285$，得到：

~~~text
maple{b4c0n_b3nd5_5p4c3t1m3_2_5f54606d7a6bfe3f}
~~~

## 方法总结

过场动画常被错误地当成纯渲染状态，但若输入、物理和坐标迁移仍同时更新，就会产生可积累的位移漏洞。审查游戏状态机时应检查每个 transition 是否冻结角色、何时切换关卡、传送使用绝对坐标还是增量向量，以及这些更新的先后顺序。
