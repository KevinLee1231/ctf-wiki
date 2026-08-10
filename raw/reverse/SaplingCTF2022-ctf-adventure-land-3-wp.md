# CTF Adventure Land 3

## 题目简述

第三题要求完成同一款游戏且用时严格小于 1700 帧。完成时间不是现实秒数，而是 Input.pos，即 checker 消耗的 replay 帧数。需要按源码分析地图碰撞、跳跃、尖刺与关卡路线，删去无效等待并走更短路径。

## 解题过程

Flag.py 的阈值顺序为 completionTimes = [1285,1410,1700]，检查时倒序处理，因此第三题条件是 time < 1700。优化时应直接统计 replay 行数，并注意胜利触发还会消耗当前帧。

官方 1652-fast-track-walk-spikes.replay 展示了可行策略：大多数平地持续按方向键，把跳跃压缩为上键从未按下到按下的一帧边沿；在尖刺区域依据 Tile.py 中独立的 solid collRect 与一像素 damageRect 安排跳跃和接触，从更短通道通过，而不是绕路逐个搬运 orb。每次改动后用 checker 完整重放，避免只在视觉模式中看似成功。

replay 的按键可直接程序化拼接：

~~~python
UP, DOWN, LEFT, RIGHT, SPACE = 1, 2, 4, 8, 16

frames = []
frames += [RIGHT] * 15
frames += [UP | RIGHT]
frames += [RIGHT] * 35
# 继续按地图位置拼接跳跃、转向和交互帧

with open("replay.txt", "w") as out:
    out.writelines(f"{value:05b}\n" for value in frames)
    out.write("\n")
~~~

完成时间低于 1700 后，服务器除第二题外继续输出：

~~~text
maple{m4pl3_b4c0n_f4573r_7h4n_299792458_m_p3r_5}
~~~

## 方法总结

游戏 speedrun 题要把画面操作转成确定性状态机问题：帧数、输入边沿、碰撞矩形和过关状态比“手感”更可复现。官方 replay 名称只是线索，真正验收条件来自 Flag.py 的严格不等式。优化应一次只改变一段，并在原 checker 中回归整条路线。
