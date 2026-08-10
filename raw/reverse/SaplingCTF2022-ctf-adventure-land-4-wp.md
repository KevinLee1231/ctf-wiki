# CTF Adventure Land 4

## 题目简述

第四题把门槛收紧到严格小于 1410 帧，普通路线和单纯操作优化不足。地图里有一种会被渲染成砖墙的 BACON_WALL tile，但 Tile.isSolid 的判断遗漏了它；玩家可以穿过视觉上的墙，直接切掉大段绕路。

## 解题过程

Tile.py 定义：

~~~python
self.isSolid = (
    id == Tile.WALL
    or self.isOrbHolder
    or self.isSpike
    or id == Tile.INVIS_WALL
)
~~~

BACON_WALL 的颜色 (145,50,0) 有对应贴图，却不在 isSolid 条件中。Map.getCloseSolidCollRects 只返回 isSolid tile，所以碰撞器完全忽略这些砖墙。把 map.png 中 BACON_WALL 做成高对比掩码可先规划捷径；最终 replay 只需对着墙持续按方向键，原始 checker 也会允许穿过。

在第三题的 fast-track 和尖刺路线基础上加入该捷径，官方 1399-wall-hack-fast-track-walk-spikes.replay 将完成时间压到 1399，满足 $1399<1410$。服务器输出：

~~~text
maple{b4c0n_b3nd5_5p4c3t1m3_961bc8b684aefa7e}
~~~

## 方法总结

游戏画面表现与碰撞语义必须分开审查。一个 tile 有贴图并不代表它参加碰撞；决定性证据是装载后的属性和碰撞查询条件。对关卡做 tile 类型掩码，常能快速发现不可见墙、假墙、伤害区和调试通道。
