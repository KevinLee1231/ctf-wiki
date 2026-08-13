# GreyCTF2024 Greycat's Adventure - Timelock WP

## 题目简述

题目声称必须等待 50 小时，但比赛只有 24 小时。核心障碍是客户端时间推进逻辑，而不是服务器端计时；既可以用速度修改绕过等待，也可以直接从 Unity IL2CPP 元数据恢复与 timelock 对应的明文 flag。

## 解题过程

先检查游戏元数据：

```bash
strings "GreyCat's Adventure_Data/il2cpp_data/Metadata/global-metadata.dat" \
  | grep -F "grey{"
```

输出中的以下字符串明确提到等待 50 小时，因而可唯一对应到 Timelock：

```text
grey{d1d_y0u_r34lly_w4it_f1fty_h0ur5???:thinking:}
```

题目要求 flag 使用小写，元数据中的字符串已经满足这一约束。

按预期动态路线复现时，可对游戏进程启用速度修改，把时间倍率大幅提高。因为倒计时由本地客户端推进，游戏观察到累计时间达到阈值后就会显示同一 flag，不需要真实等待 50 小时。

## 方法总结

看到远大于赛时的计时要求，应先判断计时权威是在服务端还是客户端。本题完全由本地 Unity 游戏控制，因此速度修改能改变结果；同时，明文元数据又提供了更短的静态恢复路径。两条路线相互验证了结论。
