# cgame

## 题目简述

附件是一个终端迷宫游戏。地图有 8 个房间，物品表中混有无法出现在有效房间范围内的干扰项。玩家收集物品时，程序把该物品的 64 字节 data 按 offset 异或进 156 字节 itemdata；收集恰好 30 个有效物品后，itemdata 被显示为 flag 字符画。

## 解题过程

可以按 WASD、门和 climb 机制实际探索，但静态恢复更直接。反编译出 items 数组及常量：

~~~text
item_total = 40
item_req   = 30
flaglen    = 156
valid rid  = 0..7
~~~

update_room 只会放置同时满足房间编号有效，且 x、y 位于 ITEM_L ≤ x < ITEM_R、ITEM_U ≤ y < ITEM_D 的物品。筛出这 30 项后，按 update_move 的原逻辑累积：

~~~python
buf = bytearray(156)
for item in valid_items:
    for j, b in enumerate(item.data):
        k = item.offset + j
        if 0 <= k < len(buf):
            buf[k] ^= b
print(buf.decode())
~~~

解出的字符画包含：

~~~text
maple{saving_the_environment_1092899062}
~~~

## 方法总结

游戏外壳不等于必须自动寻路。先找胜利条件和最终显示缓冲区，再追踪哪些对象能影响它，常能把问题化为静态数据恢复。本题的 10 个干扰物品由坐标、房间号或 offset 条件排除，必须复刻运行时边界，不能简单异或全部 40 项。
