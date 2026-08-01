# BYUCTF 2023 - Legoclones 2

## 题目简述

Legoclones 只是后来接管 Clone Trooper Wiki，题目要求找出最初创建者的用户名。

## 解题过程

站点首页不直接显示创始人，需要检索历史叙述和旧讨论。两个可相互校验的公开来源是：

- [Blyndblitz 编写的 617th Battalion 历史](https://clonetrooper.fandom.com/wiki/User:Blyndblitz/617th_History)；
- [Star Wars Fans Wiki 的 Clone Trooper Wiki 站史](https://starwarsfans.fandom.com/wiki/Clone_Trooper_Wiki)。

两者都把原始创始人指向 `Anomanzor`。其中一处历史文本误拼为 `Anonmazor`，比赛也接受该拼写：

```text
byuctf{Anomanzor}
```

## 方法总结

“创建者”与“长期管理员”不是同一身份。应优先查站史、早期讨论和创建记录，并记录来源间的拼写差异；多个独立历史页面比单个用户自述更可靠。
